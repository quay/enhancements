---
title: STS/CCO Integration for AWS S3 Authentication
authors:
  - "@sudipshil9862"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-08-31
last-updated: 2026-08-31
status: provisional
see-also:
  - "/enhancements/template.md"
---

# STS/CCO Integration for AWS S3 Authentication

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Integrate the Quay operator with OpenShift's Cloud Credential Operator (CCO)
CredentialsRequest flow so that Quay application pods on STS-enabled clusters
(ROSA, OSD) can authenticate to AWS S3 using short-lived tokens via
`AssumeRoleWithWebIdentity` instead of static IAM access keys.

The operator follows the standardized OLM + CCO `CredentialsRequest` flow. When
an admin provides an IAM role ARN during operator installation, the operator
creates a CredentialsRequest, CCO provisions a Secret containing a credentials
file, and the operator mounts that Secret into quay-app pods so boto3
transparently assumes the role.

## Motivation

Customers on ROSA/OSD-AWS clusters enforce IAM-role-only security policies
that prohibit static IAM user keys. Without this feature, those customers
cannot use real AWS S3 for Quay storage. Red Hat platform strategy mandates
all OLM operators capable of cloud API integration adopt the CCO
CredentialsRequest flow.

### Goals

- Enable Quay operator to authenticate to AWS S3 via short-lived STS tokens
  on STS-enabled OpenShift clusters using the standard CCO CredentialsRequest
  flow.
- Gracefully fall back to regular operations when no role ARN is provided.
- Degrade with clear status conditions when the role ARN is provided but CCO
  does not reconcile the CredentialsRequest (e.g., on clusters older than
  OCP 4.14).
- Document the specific IAM permissions required when integrating with AWS
  using STS and provide instructions to create the necessary IAM role.
- Support RHEL-based (non-operator) Quay deployments with documentation on
  how to supply the required role for boto3's `assume_role` flow.

### Non-Goals

- Azure WIF and GCP WIF support — tracked separately.
- Managed ObjectStorage (NooBaa) — NooBaa manages its own credentials.
- Changes to the Quay application code — boto3's standard credential chain
  reads `AWS_SHARED_CREDENTIALS_FILE` transparently.
- Support for OCP versions older than 4.14.

## Proposal

### User Stories

#### Story 1 — STS-enabled cluster installation

As a cluster admin on a ROSA/OSD cluster, I want to install the Quay operator
from OperatorHub by providing my IAM role ARN in the console UI, so that Quay
uses short-lived STS tokens for S3 access without static IAM credentials.

#### Story 2 — Graceful fallback on non-STS clusters

As a cluster admin on a standard OpenShift cluster without STS, I want the
Quay operator to work normally when no role ARN is provided, with no change
to existing behavior.

#### Story 3 — Clear error reporting

As a cluster admin, when I provide a role ARN but CCO cannot provision
credentials (e.g., on OCP < 4.14 or with a misconfigured CCO), I want to see
clear status conditions on the QuayRegistry resource explaining what is wrong
and how to fix it.

#### Story 4 — Credential conflict detection

As a cluster admin, if my storage config contains static AWS keys while a role
ARN is also set, I want the operator to block rollout and tell me to remove
the static credentials, so I do not end up in an ambiguous credential state.

#### Story 5 — RHEL-based Quay

As a RHEL-based Quay administrator (non-operator deployment), I want
documentation explaining how to configure STS authentication manually by
creating the AWS credentials file, setting the environment variable, and
providing a token at the expected path.

### Implementation Details/Notes/Constraints

**Constraints:**

- **OCP 4.14+ minimum** — older versions are explicitly out of scope.
- **AWS only** — Azure WIF and GCP WIF tracked separately.
- **`managed: false` ObjectStorage only** — NooBaa manages its own credentials
  when storage is managed.
- **No Quay app changes required** — boto3's standard credential chain reads
  `AWS_SHARED_CREDENTIALS_FILE` transparently. Verified against quay/quay
  master: `S3Storage.__init__` accepts `s3_access_key=None,
  s3_secret_key=None` as defaults; when both are `None`, boto3 falls through
  to its default credential chain.
- **Runtime CredentialsRequest creation** — the operator creates the
  CredentialsRequest at runtime (not shipped in the OLM bundle).
- **CredentialsRequest in registry namespace** — enables ownerReference-based
  garbage collection tied to the QuayRegistry CR.
- **cloudTokenPath**:
  `/var/run/secrets/openshift/serviceaccount/token` — the OpenShift-specific
  path with audience `openshift`.

**User workflow:**

1. Admin installs the Quay operator from OperatorHub, providing their IAM role
   ARN in the console UI (or via `ROLEARN` env var in the Subscription).
2. Operator detects the STS-capable cluster, creates a `CredentialsRequest`
   with the role ARN.
3. CCO provisions a Secret containing a credentials file (role ARN + web
   identity token path).
4. Operator mounts that Secret into `quay-app` pods and sets
   `AWS_SHARED_CREDENTIALS_FILE`.
5. boto3 in Quay transparently calls `AssumeRoleWithWebIdentity` and refreshes
   temporary credentials.

### Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| CCO fails to provision credentials silently | Operator sets `RolloutBlocked` condition with escalating reasons (`Pending` → `NotProvisioned` after 5-minute timeout) |
| Admin provides both static keys and role ARN | Operator detects conflict, blocks rollout with `ConflictingCredentials` condition and actionable message |
| CredentialsRequest CRD not present on cluster | Operator detects missing CRD (like existing NooBaa detection) and sets `RolloutBlocked` indicating OCP 4.14+ is required |
| Admin provides ROLEARN but ObjectStorage is managed | Operator logs a warning and skips STS path — does not block |

## Design Details

### Section 1: STS Detection Chain

Added to `controllers/quay/features.go` as `checkSTSCapability()`, following
the pattern of `checkObjectBucketClaimsAvailable()` and
`checkMonitoringAvailable()`.

**New context fields** in `pkg/context/context.go`:

```go
StorageSTSEnabled         bool
STSRoleARN                string
STSCredentialSecretName   string
STSCredentialProvisioned  bool
```

**Detection logic** (called early in the reconcile loop, after object storage
checks):

1. Read `ROLEARN` from `os.Getenv("ROLEARN")` — if empty, return immediately
   (graceful fallback).
2. Check if ObjectStorage is `managed: true` — if so, log warning and return.
3. Check if `configBundleSecret` contains static AWS keys
   (`s3_access_key`/`s3_secret_key` in `DISTRIBUTED_STORAGE_CONFIG`) — if so,
   set `RolloutBlocked` with `ConflictingCredentials` reason.
4. Verify CredentialsRequest CRD exists on cluster (unstructured list, like
   `ObjectBucketClaim` detection) — if not, set `RolloutBlocked` indicating
   CCO/OCP 4.14+ required.
5. Set `qctx.StorageSTSEnabled = true` and `qctx.STSRoleARN = roleARN`.

### Section 2: CredentialsRequest Lifecycle

Logic lives in `controllers/quay/quayregistry_controller.go` as
`ensureCredentialsRequest()`, called from the reconcile loop after
`checkSTSCapability()` populates the context.

**CredentialsRequest object spec:**

- **Name**: `<quayregistry-name>-aws-credentials`
- **Namespace**: Same as QuayRegistry
- **OwnerReference**: Set to QuayRegistry CR (enables GC on CR deletion)
- **ProviderSpec**: AWS provider with `s3:*` statement entry and the role ARN
- **SecretRef**: `<quayregistry-name>-aws-sts-credentials` in the registry
  namespace
- **ServiceAccountNames**: The quay-app service account
- **CloudTokenPath**:
  `/var/run/secrets/openshift/serviceaccount/token`

**Reconcile flow:**

1. If `qctx.StorageSTSEnabled` is false, skip entirely.
2. Build the desired CredentialsRequest spec.
3. Get or create the CredentialsRequest, updating if the role ARN changed.
4. Check for the CCO-provisioned Secret:
   - If found with `credentials` key: mark provisioned, continue to Inflate.
   - If not found: set pending condition, requeue after 10 seconds.
5. If Secret hasn't appeared after 5 minutes (based on CredentialsRequest
   `creationTimestamp`), escalate to `RolloutBlocked` with
   `CredentialRequestNotProvisioned`.

### Section 3: Volume and Environment Injection

Logic lives in `pkg/middleware/middleware.go` as `applySTSCredentials()`,
called from the `Process()` function alongside `applyPostgresTLS()` and
`applyClairDBTLS()`. Only runs on `quay-app` and `quay-mirror` deployments.

**Three things are mounted into quay-app/quay-mirror pods:**

1. **CCO-provisioned credentials Secret** — mounted read-only at `/aws-sts`
2. **Bound service account token** — projected volume mounted at
   `/var/run/secrets/openshift/serviceaccount` with audience `openshift`
3. **Environment variable** — `AWS_SHARED_CREDENTIALS_FILE=/aws-sts/credentials`

**Guard conditions:**
- Only applies when `qctx.StorageSTSEnabled && qctx.STSCredentialProvisioned`
- Only applies to `quay-app` and `quay-mirror` deployments
- Does not apply to clair-app, postgres, redis, or other deployments

### Section 4: CSV Changes

Two modifications to the ClusterServiceVersion:

1. **Annotation**: Set `features.operators.openshift.io/token-auth-aws: "true"`
   so OperatorHub shows the role ARN input field during installation. OLM
   injects the admin-provided ARN as the `ROLEARN` env var on the operator pod
   via the Subscription config.

2. **clusterPermissions**: Add RBAC for CredentialsRequest CRUD:
   ```yaml
   - apiGroups:
     - "cloudcredential.openshift.io"
     resources:
     - credentialsrequests
     verbs:
     - create
     - delete
     - get
     - list
     - patch
     - update
     - watch
   ```

### Section 5: Condition Reporting

New condition reasons added to `apis/quay/v1/quayregistry_types.go`:

| Condition | Type | Trigger | Cleared when |
|-----------|------|---------|-------------|
| `CredentialRequestPending` | `RolloutBlocked` | CredentialsRequest created but CCO Secret not yet provisioned (< 5 min) | Secret appears |
| `CredentialRequestNotProvisioned` | `RolloutBlocked` | CCO Secret still missing after 5 minutes | Secret appears |
| `ConflictingCredentials` | `RolloutBlocked` | ROLEARN set + static `s3_access_key`/`s3_secret_key` in configBundleSecret | Admin removes static keys from config |

All three use the existing `ConditionTypeRolloutBlocked` type and follow the
same pattern as `ConditionReasonConfigInvalid` and
`ConditionReasonComponentCreationFailed`.

### Relevant Codebase Files

| File | Role | What changes |
|------|------|-------------|
| `pkg/context/context.go` | Reconcile-scoped state | Add STS context fields |
| `controllers/quay/features.go` | Feature detection | Add `checkSTSCapability()` |
| `controllers/quay/quayregistry_controller.go` | Main reconcile loop | Add CredentialsRequest lifecycle, conditions |
| `pkg/middleware/middleware.go` | Deployment mutation | Add `applySTSCredentials()` |
| `apis/quay/v1/quayregistry_types.go` | Condition types | Add new condition reasons |
| `bundle/manifests/quay-operator.clusterserviceversion.yaml` | OLM metadata | Update annotation, add clusterPermissions |

### Test Plan

- **Unit tests**: Test `checkSTSCapability()` detection logic across all
  branches (no ROLEARN, managed storage, static key conflict, missing CRD,
  happy path).
- **Unit tests**: Test `ensureCredentialsRequest()` create/update/wait logic
  with mock client.
- **Unit tests**: Test `applySTSCredentials()` volume and env injection on
  quay-app/quay-mirror deployments, and verify it is skipped for other
  deployments.
- **Integration tests**: Test full reconcile loop with a mock CCO (fake
  CredentialsRequest CRD, fake Secret provisioning) to verify end-to-end
  context flow from detection through injection.
- **E2E tests**: On a real STS-enabled cluster (ROSA), verify:
  - Operator creates CredentialsRequest with correct spec
  - CCO provisions the expected Secret
  - quay-app pods have the correct volume mounts and env var
  - Quay can read/write objects to S3 without static credentials
  - Removing ROLEARN and redeploying reverts to standard behavior

### Graduation Criteria

#### Dev Preview

- Core STS detection and CredentialsRequest lifecycle implemented
- Volume and environment injection working on quay-app pods
- Unit test coverage for all detection branches and injection logic

#### Tech Preview

- End-to-end testing on ROSA clusters
- Documentation for IAM role setup, installation workflow, and RHEL-based
  deployments
- Condition reporting verified in console UI
- quay-mirror deployment also receives STS credentials

#### GA

- Sufficient production feedback from STS-enabled cluster deployments
- Upgrade/downgrade testing (operator upgrade with existing CredentialsRequest)
- Available by default when ROLEARN is provided
- IAM permission documentation reviewed and finalized

### API Design

No new API endpoints are introduced. The feature is configured via:
- `ROLEARN` environment variable on the operator pod (set by OLM from
  Subscription config)
- Existing `QuayRegistry` status conditions for reporting

The operator creates a `CredentialsRequest` custom resource (owned by
OpenShift's CCO, not Quay) in the registry namespace.

### Upgrade / Downgrade Strategy

- **Upgrade**: Existing clusters without ROLEARN see no change. Clusters where
  an admin subsequently sets ROLEARN via Subscription config will trigger the
  STS flow on the next reconcile.
- **Downgrade**: If the operator is downgraded to a version without STS
  support, the CredentialsRequest remains orphaned (ownerReference still
  points to QuayRegistry). The admin should manually clean up the
  CredentialsRequest. The quay-app pods will revert to whatever credential
  method is configured in the storage config.

### Version Skew Strategy

- The operator requires OCP 4.14+ for CCO STS support. On older clusters, the
  CredentialsRequest CRD will not exist and the operator will set a
  `RolloutBlocked` condition.
- The feature has no cross-component version skew concerns within Quay itself
  — it is entirely operator-side configuration.

## Implementation History

- 2026-08-31: Initial enhancement proposal

## Drawbacks

- Adds a new external dependency on the `cloud-credential-operator` Go module
  for CredentialsRequest types.
- The 5-minute timeout for CCO Secret provisioning is a heuristic — in edge
  cases CCO may take longer, leading to a premature `NotProvisioned` condition
  that resolves on its own.

## Alternatives

- **Bundle-shipped CredentialsRequest** (Alternative 2 from STS guide): The
  CredentialsRequest is shipped in the OLM bundle instead of created at
  runtime. This was rejected because it requires admin pre-work with `ccoctl`
  and is less user-friendly.
- **Existing `STSS3Storage` driver**: Quay has an existing `STSS3Storage`
  class that uses the older `assume_role` flow with static IAM user keys.
  This was not used because the standard `S3Storage` driver with
  `AWS_SHARED_CREDENTIALS_FILE` is the correct approach for the CCO/web-identity
  flow and requires no Quay app changes.
