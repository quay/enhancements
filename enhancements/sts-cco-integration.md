---
title: STS/CCO Integration for AWS S3 Authentication
authors:
  - "@sudipshil9862"
reviewers:
  - "@Marcusk19"
approvers:
  - TBD
creation-date: 2026-08-31
last-updated: 2026-10-05
status: implemented
see-also:
  - "https://github.com/quay/quay-operator/pull/1324"
  - "https://github.com/openshift/release/pull/85698"
---

# STS/CCO Integration for AWS S3 Authentication

## Release Signoff Checklist

- [x] Enhancement is implementable
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined and implemented
- [x] Graduation criteria for dev preview, tech preview, and GA are defined

## Summary

The Quay Operator supports OpenShift Cloud Credential Operator (CCO)
`CredentialsRequest` integration so that Quay application and repository-mirror
pods can access an existing AWS S3 bucket with short-lived credentials instead
of storing long-lived AWS access keys in the Quay configuration.

An administrator creates the S3 bucket, bucket-scoped IAM policy, IAM role, and
OIDC trust relationship. The administrator supplies the role ARN to the Quay
Operator through `ROLEARN` and configures unmanaged `S3Storage` without static
keys. The Operator validates the environment, creates a `CredentialsRequest`,
waits for and validates the profile Secret produced by CCO, and mounts that
profile together with a projected OpenShift service-account token into the Quay
app and mirror pods. The AWS SDK in those workloads exchanges the token for
temporary credentials through `AssumeRoleWithWebIdentity` and uses them to
access S3.

The Quay Operator configures this identity flow but never calls AWS STS or S3
itself.

Implementation and live validation are available in:

- [quay/quay-operator#1324](https://github.com/quay/quay-operator/pull/1324)
- [openshift/release#85698](https://github.com/openshift/release/pull/85698)
- [quay/quay-operator#1350](https://github.com/quay/quay-operator/pull/1350)
  (post-merge STS validation)

## Motivation

Customers running Quay on ROSA, OSD-AWS, and other STS-enabled OpenShift
clusters may prohibit static IAM user keys. Before this enhancement, an
external AWS S3 configuration normally required long-lived access and secret
keys in the Quay config bundle. Those keys introduce storage, rotation, and
security-policy concerns.

OpenShift provides a standard workload-identity flow in which a pod presents a
short-lived OpenShift service-account token to AWS STS. AWS verifies the token
through the cluster's registered OIDC provider and returns temporary
credentials. Integrating Quay with CCO follows this platform pattern and removes
the need to store static AWS keys in Quay configuration.

### Goals

- Enable Quay app and mirror workloads to access external AWS S3 with
  short-lived STS credentials.
- Follow the standard OLM and CCO `CredentialsRequest` pattern.
- Preserve existing behavior when `ROLEARN` is not configured.
- Support only applicable unmanaged `S3Storage` entries.
- Reject ambiguous configurations containing both `ROLEARN` and static S3
  credentials.
- Report actionable pending and failure conditions on `QuayRegistry`.
- Restrict generated S3 permission statements to configured buckets and the
  operations Quay requires.
- Document IAM, OIDC, Operator installation, Quay configuration, verification,
  and standalone RHEL-based Quay requirements.
- Validate the complete flow on a real, isolated AWS OpenShift cluster.

### Non-Goals

- Creating a customer's production S3 bucket, IAM role, IAM policy, or AWS OIDC
  provider.
- Giving the Quay Operator itself AWS credentials or direct S3 access.
- Supporting managed NooBaa/ODF object storage through this flow.
- Supporting Azure or Google workload identity.
- Using the legacy Quay `STSS3Storage` driver.
- Providing static AWS keys as an STS fallback.
- Supporting OpenShift versions older than 4.14.

## Proposal

### User Stories

#### Story 1 — STS-enabled OpenShift installation

As a cluster administrator, I want to configure the Quay Operator with an IAM
role ARN and an existing S3 bucket so that Quay uses short-lived AWS
credentials without storing static access keys.

#### Story 2 — Backward-compatible default

As an existing Quay administrator, I want deployments without `ROLEARN` to
continue using their current storage behavior without any STS resources or
workload changes.

#### Story 3 — Clear prerequisite failures

As a cluster administrator, I want the Operator to explain when the cluster is
not on AWS, CCO is not in Manual mode, the OIDC issuer is missing, the
CredentialsRequest API is unavailable, or CCO cannot provision the current
request.

#### Story 4 — Credential conflict detection

As a cluster administrator, I want the Operator to block rollout when an
applicable `S3Storage` entry contains static keys while `ROLEARN` is enabled, so
that Quay does not start with an ambiguous credential source.

#### Story 5 — RHEL-based Quay

As a standalone Quay administrator, I want documentation explaining how to
provide a web-identity token and shared AWS profile without the Operator or CCO.

### Prerequisites and Ownership

The AWS or cluster administrator creates and owns:

- the external S3 bucket;
- a policy limited to that bucket;
- an IAM role with that policy;
- a trust policy for the cluster's AWS OIDC provider;
- the OLM `ROLEARN` configuration;
- the credential-free Quay `S3Storage` configuration.

The role trust policy permits `sts:AssumeRoleWithWebIdentity` only for the
expected service-account identity:

```text
aud = openshift
sub = system:serviceaccount:<namespace>:<quayregistry-name>-quay-app
```

CCO runs in `Manual` credentials mode for this design. It processes the
`CredentialsRequest` and creates a Kubernetes profile Secret, but it does not
create the AWS IAM role or policy.

### User Workflow

1. The administrator creates the S3 bucket, IAM policy, IAM role, and OIDC trust
   relationship.
2. The administrator sets `ROLEARN` in the Quay Operator Subscription.
3. The administrator configures `objectstorage` as unmanaged and provides an
   `S3Storage` entry with a bucket and no static keys.
4. The Operator validates the configuration and cluster capabilities.
5. The Operator creates or updates the `CredentialsRequest`.
6. CCO creates a profile Secret containing `role_arn` and
   `web_identity_token_file`.
7. The Operator validates the current CCO request generation and profile.
8. The Operator configures Quay app and mirror Deployments with the profile and
   projected service-account token.
9. Quay calls AWS STS and uses the returned temporary credentials for S3.
10. The AWS SDK refreshes temporary credentials as they approach expiration.

### Constraints

- OpenShift 4.14 or later.
- AWS infrastructure platform.
- CCO `Manual` mode.
- A non-empty OpenShift service-account issuer and corresponding AWS IAM OIDC
  provider.
- `objectstorage` set to `managed: false`.
- Standard `S3Storage` with one or more concrete `s3_bucket` values.
- No `s3_access_key` or `s3_secret_key` in applicable `S3Storage` entries.
- Quay 3.17 or later, or a build containing the S3 default-credential-chain
  fix needed to resolve the CCO web-identity profile.
- `cloudTokenPath` set to
  `/var/run/secrets/openshift/serviceaccount/token` with audience `openshift`.

## Design Details

### STS Capability Detection

`controllers/quay/features.go` implements `checkSTSCapability()`.

The reconcile context includes:

```go
StorageSTSEnabled        bool
STSRoleARN               string
STSStorageBuckets        []string
STSCredentialSecretName  string
STSCredentialProvisioned bool
```

When `ROLEARN` is set, the Operator checks:

1. Object storage is unmanaged.
2. The flattened config contains at least one standard `S3Storage` backend.
3. Applicable S3 entries contain concrete bucket names.
4. Applicable S3 entries do not contain static access or secret keys.
5. The `CredentialsRequest` API was discovered through the RESTMapper when the
   controller started.
6. The cluster infrastructure platform is AWS.
7. The cluster `CloudCredential` mode is `Manual`.
8. The cluster has a configured service-account issuer.

Static credentials belonging to unrelated storage drivers do not enable or
block the S3 STS flow. Managed storage and configurations without an applicable
`S3Storage` backend skip STS rather than changing normal behavior.

### CredentialsRequest Shape

The request is created in the `QuayRegistry` namespace with the name:

```text
<quayregistry-name>-aws-credentials
```

It contains:

- the configured role ARN;
- the Quay app service-account name;
- the projected-token file path;
- the requested output Secret name and namespace;
- bucket-level and object-level S3 statements for each configured bucket;
- a `QuayRegistry` owner reference.

The output Secret name is:

```text
<quayregistry-name>-aws-sts-credentials
```

Bucket-level actions are:

```text
s3:ListBucket
s3:GetBucketLocation
s3:ListBucketMultipartUploads
```

Object-level actions are:

```text
s3:GetObject
s3:PutObject
s3:DeleteObject
s3:AbortMultipartUpload
s3:ListMultipartUploadParts
```

Resources are limited to:

```text
arn:<partition>:s3:::<bucket>
arn:<partition>:s3:::<bucket>/*
```

The AWS partition is derived from the role ARN. Wildcard bucket names are
rejected.

The implementation uses local wire types in `pkg/credentialsrequest/` that
mirror the required CCO API fields. It intentionally avoids importing CCO's
internal Go API and its incompatible transitive Kubernetes dependencies.

### CredentialsRequest Lifecycle

The Operator uses only these verbs for `CredentialsRequest`:

```text
create, get, update, delete
```

The Operator:

1. Creates the request when it does not exist.
2. Refuses to adopt a same-name request not owned by the current
   `QuayRegistry`.
3. Updates the specification when the role, buckets, service account, output
   Secret, or token path changes.
4. Waits for `status.provisioned=true`.
5. Requires `status.lastSyncGeneration` to match the current generation.
6. Waits for the output Secret.
7. Escalates the condition after the five-minute provisioning timeout.

A request timestamp annotation is updated when the desired request changes so
that the timeout applies to the current provisioning attempt.

When STS no longer applies, the Operator deletes only the same-name request
owned by the current `QuayRegistry`. The owner reference also allows Kubernetes
garbage collection when the registry is deleted.

### CCO Profile Secret

The CCO Secret contains a shared profile similar to:

```ini
[default]
role_arn = arn:aws:iam::123456789012:role/example-quay-sts
web_identity_token_file = /var/run/secrets/openshift/serviceaccount/token
```

The service-account token is not stored in this Secret. OpenShift projects it
separately into the pod.

Before rendering STS-enabled workloads, the Operator verifies:

- the `credentials` entry is non-empty;
- static `aws_access_key_id` and `aws_secret_access_key` entries are absent;
- `role_arn` exactly matches `ROLEARN`;
- `web_identity_token_file` exactly matches the expected projected path.

### Workload Configuration

STS configuration applies only to Deployments ending in `quay-app` or
`quay-mirror`.

The Operator adds:

```text
Secret volume:     aws-sts-credentials
Secret mount:      /aws-sts (read only)
Projected volume:  bound-sa-token
Token mount:       /var/run/secrets/openshift/serviceaccount (read only)
Token audience:    openshift
```

It also adds:

```text
AWS_SHARED_CREDENTIALS_FILE=/aws-sts/credentials
AWS_SDK_LOAD_CONFIG=true
```

Same-name volumes and mounts are replaced with the desired definitions instead
of being retained unchanged. This keeps reconciliation idempotent and repairs
conflicting existing definitions.

PostgreSQL, Redis, Clair, and unrelated workloads do not receive these mounts or
environment variables.

### Runtime Credential Flow

```text
OpenShift API server
    -> signs and rotates the Quay service-account JWT
Quay AWS SDK
    -> reads role_arn and the projected JWT
    -> calls AWS STS AssumeRoleWithWebIdentity
AWS STS
    -> verifies issuer, signature, audience, subject, expiration, and trust
    -> returns temporary access key, secret key, session token, and expiration
Quay AWS SDK
    -> caches and refreshes the temporary session
    -> signs S3 requests
AWS S3
    -> authorizes requests against the role and bucket policy
```

Neither CCO nor the Quay Operator receives the temporary STS credentials. They
remain in the AWS SDK credential provider inside the Quay process.

### OLM and RBAC

The CSV declares:

```yaml
features.operators.openshift.io/token-auth-aws: "true"
```

The Operator's CredentialsRequest RBAC is limited to:

```text
create;delete;get;update
```

The Operator pod does not receive a projected AWS token because the Operator
does not call AWS APIs.

### Condition Reporting

The implementation uses the existing `RolloutBlocked` condition type.

| Reason | Trigger | Resolution |
|---|---|---|
| `CredentialRequestPending` | The current request generation or output Secret is not ready before the timeout. | CCO provisions the current request and valid Secret. |
| `CredentialRequestNotProvisioned` | Provisioning exceeds five minutes or the profile is missing/invalid. | Correct CCO, role, profile, or cluster configuration. |
| `ConflictingCredentials` | An applicable `S3Storage` entry contains static keys while `ROLEARN` is set. | Remove static keys or remove `ROLEARN`. |
| `ConfigInvalid` | AWS, CCO mode, OIDC issuer, role ARN, S3, or API prerequisites are invalid. | Correct the reported prerequisite. |

Rollout remains blocked until a valid current-generation profile is available.

### Security Properties

- No static AWS access key or secret key is stored in Quay configuration.
- The CCO profile stores only the role ARN and token-file path.
- OpenShift rotates the projected service-account token.
- The IAM trust policy limits issuer, audience, namespace, and service account.
- S3 permissions are limited to configured buckets and required actions.
- Only Quay app and mirror workloads receive the profile and token.
- The Operator does not receive S3 credentials or access S3.
- RBAC follows least privilege.
- Same-name CredentialsRequests are not adopted across registries.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| CCO never provisions the request | Pending condition followed by a blocked NotProvisioned reason after five minutes. |
| Stale CCO result is used after a request change | Require `lastSyncGeneration` to match the current generation. |
| A malformed or static-key profile is mounted | Validate profile contents, role ARN, and token path before rollout. |
| Static keys and STS are configured together | Block with `ConflictingCredentials`. |
| Permissions are broader than required | Generate bucket-specific actions and document a matching IAM policy. |
| An unrelated request uses the expected name | Verify the `QuayRegistry` owner reference and refuse adoption. |
| Older Quay image cannot resolve web identity | Require Quay 3.17+ or the default-credential-chain fix. |
| CCO takes longer than five minutes | Reconciliation continues; the condition can recover when CCO eventually provisions a valid current generation. |

Security-sensitive setup is documented in `docs/sts-iam-setup.md` in the Quay
Operator repository.

## Test Plan

### Unit and Controller Tests

Tests cover:

- empty `ROLEARN` fallback;
- managed storage and non-S3 storage behavior;
- AWS platform, CCO mode, OIDC issuer, and CRD prerequisites;
- static-key conflict detection scoped to `S3Storage`;
- role ARN and AWS partition validation;
- bucket-specific CredentialsRequest construction;
- create, update, ownership, generation, timeout, validation, and cleanup paths;
- profile rejection for static keys, wrong role, or wrong token path;
- app/mirror-only injection;
- environment variables, volumes, mounts, replacement, and idempotency.

### Live E2E Test

[openshift/release#85698](https://github.com/openshift/release/pull/85698)
added the optional `ocp-latest-e2e-sts` presubmit. It uses the standard
`openshift-org-aws` cluster profile and creates an isolated AWS IPI OpenShift
cluster with CCO Manual mode and a per-cluster OIDC provider.

The job:

1. Creates the ephemeral cluster and OIDC infrastructure.
2. Creates a unique namespace, S3 bucket, IAM role, trust policy, and
   bucket-scoped inline policy.
3. Installs the PR-built Quay Operator bundle with `ROLEARN`.
4. Runs `make test-e2e-sts` from the Operator source.
5. Verifies the request, current generation, profile, token audience, workload
   mounts, and absence of static keys.
6. Pushes and pulls a real image through S3.
7. Configures repository mirroring and pulls the mirrored image.
8. Destroys the cluster, OIDC/platform infrastructure, role, policy, S3
   contents, multipart uploads, versions, and bucket.

The job does not use the shared Quay QE cluster or a Quay DEV credential
collection. Every run owns its cluster and AWS resources.

The final post-merge validation passed in
[quay/quay-operator#1350](https://github.com/quay/quay-operator/pull/1350):

```text
ci/prow/ocp-latest-e2e-sts: PASS
```

### E2E Race Fixes

Two test timing problems were corrected during live validation:

- The test now waits for the Quay app and mirror Deployments to exist before
  running `kubectl rollout status`.
- Repository automatic synchronization is scheduled one hour in the future
  before the test calls `sync-now`, preventing the background worker and manual
  trigger from racing.

### Relationship to Other E2E Jobs

- `ocp-latest-e2e` uses managed ODF/NooBaa storage and runs the general
  Chainsaw suite. It does not validate short-lived AWS credentials.
- `ocp-latest-e2e-sts` uses a Manual-mode/OIDC cluster and real external S3.
- KinD E2E cannot validate AWS, OpenShift CCO, OIDC, or STS behavior.

## Graduation Criteria

### Dev Preview

- STS capability detection and graceful fallback implemented.
- CredentialsRequest lifecycle and profile validation implemented.
- App and mirror workload injection implemented.
- Unit tests cover detection, request construction, and middleware behavior.

**Status:** Complete.

### Tech Preview

- AWS IAM and installation documentation available.
- Real ephemeral AWS CCO/STS/S3 E2E test available.
- Status and failure conditions verified.
- Live push, pull, and mirror operations pass without static credentials.

**Status:** Complete.

### GA

- Included in a supported Quay Operator release with a compatible Quay image.
- Documentation reviewed for the target release.
- Upgrade and downgrade behavior validated for supported release paths.
- Sufficient production or field feedback from STS-enabled deployments.
- Periodic or otherwise regularly exercised live STS coverage is maintained if
  CI cost permits.

## API Design

No new Quay API endpoint or `QuayRegistry` specification field is introduced.
Configuration uses:

- `ROLEARN` on the Operator Deployment, normally supplied through the OLM
  Subscription;
- existing unmanaged object-storage configuration in the config bundle;
- existing `QuayRegistry` status conditions.

The Operator creates a `cloudcredential.openshift.io/v1` CredentialsRequest.
The resource has a `QuayRegistry` owner reference; CCO watches and processes it.

## Upgrade / Downgrade Strategy

### Upgrade

Existing installations without `ROLEARN` are unchanged. To enable STS, the
administrator must create the AWS resources, remove static S3 keys, configure
`ROLEARN`, and ensure the external S3 config is unmanaged. The Operator then
creates the request and blocks rollout until CCO provisions a valid profile.

Existing owned requests are updated when their desired role, buckets, service
account, output Secret, or token path changes.

### Downgrade

Before downgrading to an Operator version without this feature, the
administrator must choose another credential mechanism. For the traditional S3
flow this means restoring static access and secret keys in the config bundle and
removing `ROLEARN` before downgrade.

An older Operator does not understand or mount the CCO profile. A remaining
CredentialsRequest continues to have a `QuayRegistry` owner reference but may
need manual removal after downgrade. The associated IAM role and S3 bucket
remain administrator-owned and are not deleted by the Operator.

## Version Skew Strategy

- OpenShift 4.14+ is required for the supported CCO API and short-lived-token
  flow.
- The cluster must expose the CredentialsRequest API, AWS infrastructure,
  Manual CCO mode, and a service-account issuer.
- Quay 3.17+ or an image containing the S3 default-credential-chain fix is
  required.
- The Operator waits for the current CredentialsRequest generation before
  changing workloads, preventing use of stale profile state during updates.
- Older kubelets or non-OpenShift Kubernetes clusters are outside the supported
  flow because the required CCO and OpenShift identity capabilities are absent.

## Implementation History

- **2026-08-31:** Initial enhancement proposal opened.
- **2026-09-24:** Ephemeral STS CI job merged through
  [openshift/release#85698](https://github.com/openshift/release/pull/85698).
- **2026-09-25:** Live CCO/STS/S3 push, pull, and mirroring validation passed on
  the feature PR.
- **2026-09-30:** Operator implementation merged through
  [quay/quay-operator#1324](https://github.com/quay/quay-operator/pull/1324).
- **2026-09-30:** Post-merge STS validation passed through
  [quay/quay-operator#1350](https://github.com/quay/quay-operator/pull/1350).

## Drawbacks

- Local CredentialsRequest wire types must remain compatible with the supported
  CCO API fields.
- `ROLEARN` is configured at Operator-installation scope. An Operator managing
  several registries requires a role trust policy covering every intended Quay
  service-account subject, or separate Operator installations with appropriate
  scope.
- The five-minute escalation threshold is a heuristic. Slow CCO provisioning
  can temporarily report `CredentialRequestNotProvisioned`, although later
  reconciliation can recover.
- Live validation creates an AWS OpenShift cluster and has greater time and
  infrastructure cost than unit or KinD tests.

## Alternatives

### Store static AWS credentials

Rejected as the primary solution because it retains long-lived secrets and does
not satisfy IAM-role-only customer policies.

### Bundle-shipped CredentialsRequest

Rejected in favor of runtime creation because bucket names, registry namespace,
service account, and role are instance-specific. Runtime reconciliation also
supports ownership, updates, status checks, and cleanup.

### Legacy `STSS3Storage`

Rejected because it represents an older assume-role flow based on static source
credentials. Standard `S3Storage` with the AWS default credential chain is the
correct web-identity integration.

### Shared Quay QE cluster for live testing

The initial CI proposal used a long-lived shared cluster and separate Quay QE
and Quay DEV credentials. It was replaced by an ephemeral cluster because
shared OLM installations, cluster-wide CRDs, OIDC lifecycle, concurrent runs,
and cleanup created unnecessary coupling and risk.

## Infrastructure Needed

The required CI infrastructure is implemented in
[openshift/release#85698](https://github.com/openshift/release/pull/85698):

- optional `ocp-latest-e2e-sts` presubmit;
- standard `openshift-org-aws` cluster profile;
- ephemeral Manual-mode/OIDC AWS OpenShift cluster;
- run-owned S3 and IAM resources;
- PR-built Operator installation;
- complete post-job cluster and AWS cleanup.

No shared Quay QE kubeconfig or Quay DEV AWS credential collection is required.
