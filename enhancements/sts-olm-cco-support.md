---
title: Standardized STS Configuration via OLM and CCO for Quay on OpenShift
authors:
  - "@dmesser"
  - "@doconnor"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2023-07-19
last-updated: 2023-07-19
status: implementable
see-also:
  - "https://issues.redhat.com/browse/OCPSTRAT-171"
  - "https://issues.redhat.com/browse/OCPSTRAT-6"
  - "https://issues.redhat.com/browse/PROJQUAY-7729"
---

# Standardized STS Configuration via OLM and CCO for Quay on OpenShift

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

1. Should the Quay operator create and own the `CredentialRequest` CR, or should it be pre-created by the user during installation?
2. How does the operator detect whether it is running on an STS-enabled cluster vs. a standard AWS cluster at runtime?
3. What is the exact set of IAM actions required by Quay for object storage access? Should the operator publish a managed IAM policy document?
4. For RHEL-based (non-OCP) deployments, is a separate configuration guide sufficient or is tooling needed to simplify role injection into the pod?

## Summary

AWS STS (Security Token Service) based authentication eliminates the need for static, long-lived AWS access keys by exchanging a Kubernetes-projected service account token for short-lived IAM credentials via the `AssumeRoleWithWebIdentity` flow. OpenShift's Cloud Credential Operator (CCO) standardizes this across all OLM-managed operators through the `CredentialRequest` API.

This enhancement integrates the Quay operator with the CCO `CredentialRequest` flow so that Quay on STS-enabled OpenShift clusters can authenticate to AWS object storage (and other AWS APIs) without static credentials. The implementation follows the pattern defined in OCPSTRAT-171 / OCPSTRAT-6 so administrators get the same experience configuring Quay's operator as they do with any other CCO-integrated OLM operator.

## Motivation

Quay uses AWS S3 (or S3-compatible object storage via OpenShift Data Foundation/RHOCS) as its primary blob storage backend. Today, the operator configures Quay with static `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` credentials sourced from an ObjectBucketClaim Secret. This approach has several drawbacks:

- **Static credentials** are a security liability: they do not rotate automatically and must be managed out-of-band.
- **ROSA and OSD clusters** commonly enforce IAM-role-only policies and prohibit static IAM user keys entirely, making Quay incompatible with these environments.
- **Inconsistency** across OLM operators: other operators (e.g., RHACM, ODF) have already adopted the CCO `CredentialRequest` flow. Quay's divergence creates operational friction for administrators familiar with the standard pattern.
- **Customer demand**: Elevance Health (Anthem) and other strategic accounts require STS-based auth as a hard requirement for deploying Quay on ROSA.

Red Hat's platform strategy (OCPSTRAT-6) mandates that all OLM-managed operators capable of integrating with cloud provider APIs adopt the CCO-based `CredentialRequest` flow. Quay has been explicitly identified as a target operator.

### Goals

- Implement the standardized CCO `CredentialRequest` flow in the Quay operator for AWS STS.
- Enable Quay to authenticate to AWS object storage using short-lived STS credentials projected via service account token.
- Gracefully fall back to the existing static-credential path when no IAM role ARN is provided (preserving backwards compatibility for non-STS environments).
- Degrade the `QuayRegistry` with an informative condition when a role ARN is configured but CCO fails to reconcile the `CredentialRequest` (e.g. on OCP < 4.14 or on a non-STS cluster).
- Document the required IAM permissions and provide easy-to-follow instructions for creating and attaching the IAM role.
- Provide instructions for RHEL-based Quay deployments to supply the IAM role for boto's `assume_role()` flow.
- Annotate the Quay CSV with `features.operators.openshift.io/token-auth-aws: "true"` so the OCP console and OperatorHub can discover and surface the capability.

### Non-Goals

- Support for OCP versions older than 4.14 (the minimum version at which the standardized CCO flow is available).
- STS support for non-AWS cloud providers (Azure Workload Identity, GCP WIF are tracked separately in PROJQUAY-7729).
- Automatic IAM role or policy creation; the operator will document requirements but not provision IAM resources in the customer's AWS account.
- Changes to the Quay application itself (`quay/quay`); all changes are confined to the operator.

## Proposal

### Overview

The CCO `CredentialRequest` flow works as follows:

1. The Quay operator creates a `CredentialRequest` CR in its own namespace, specifying the required AWS IAM permissions (`s3:GetObject`, `s3:PutObject`, etc.) and the service account to bind.
2. CCO reads the `CredentialRequest`, calls `sts:AssumeRoleWithWebIdentity` using the operator service account's projected OIDC token, and writes short-lived credentials into a `Secret` in the operator namespace.
3. The Quay operator reads that `Secret` and injects the STS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) into Quay's `config.yaml` distributed storage configuration.
4. Quay's boto-based storage driver uses the credentials to access S3. CCO handles automatic rotation before expiry.

The operator must know the customer's IAM role ARN to include in the `CredentialRequest`. This ARN is provided by the administrator as an annotation on the `QuayRegistry` resource (following the pattern used by other CCO-integrated operators).

### User Stories

#### Story 1 — ROSA Administrator installs Quay without static credentials

As a ROSA cluster administrator whose security policy prohibits static IAM user keys, I want to install Quay via OperatorHub, provide my pre-created IAM role ARN as an annotation on the `QuayRegistry` resource, and have the operator automatically configure Quay to use STS credentials — without any static keys appearing in Secrets.

#### Story 2 — Existing Quay installation on OCP retains static credential behavior

As an OCP cluster administrator running Quay with existing static S3 credentials configured through an ObjectBucketClaim, I want to upgrade the Quay operator without any change in behavior; my static credentials should continue to work unless I explicitly opt in to STS.

#### Story 3 — Operator surfaces a clear error when STS configuration is incomplete

As a ROSA cluster administrator, if I annotate the `QuayRegistry` with a role ARN but CCO cannot reconcile the `CredentialRequest` (e.g., the ARN is wrong or the OIDC provider is not configured), I want the `QuayRegistry` to report a `Degraded` condition with a human-readable message explaining what went wrong.

### Implementation Details

#### 1. IAM Role ARN Input

The administrator annotates the `QuayRegistry` with the role ARN before or after creation:

```yaml
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: example-registry
  annotations:
    quay-operator/aws-sts-role-arn: "arn:aws:iam::123456789012:role/quay-s3-role"
spec:
  components:
    - kind: ObjectStorage
      managed: true
```

The operator reads this annotation during reconciliation. If it is absent, the operator follows the existing static-credential path (no change to current behavior).

#### 2. CredentialRequest Lifecycle

When a role ARN annotation is present, the operator creates or updates a `CredentialRequest` in its own namespace:

```yaml
apiVersion: cloudcredential.openshift.io/v1
kind: CredentialRequest
metadata:
  name: quay-operator-aws
  namespace: openshift-operators
spec:
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: AWSProviderSpec
    statementEntries:
      - effect: Allow
        action:
          - s3:GetObject
          - s3:PutObject
          - s3:DeleteObject
          - s3:ListBucket
          - s3:GetBucketLocation
        resource: "arn:aws:s3:::${BUCKET_NAME}/*"
      - effect: Allow
        action:
          - s3:ListBucket
          - s3:GetBucketLocation
        resource: "arn:aws:s3:::${BUCKET_NAME}"
  secretRef:
    name: quay-aws-sts-credentials
    namespace: openshift-operators
  serviceAccountNames:
    - quay-operator
```

CCO populates `quay-aws-sts-credentials` with:
- `credentials` (AWS credentials file format with `role_arn` and `web_identity_token_file`)
- `aws_access_key_id` / `aws_secret_access_key` / `aws_session_token` (short-lived)

#### 3. Operator Reconciliation Changes

In `controllers/quay/features.go` (`checkObjectBucketClaimsAvailable`):
- After extracting `StorageHostname` and `StorageBucketName` from the ObjectBucketClaim as today, check whether the STS ARN annotation is present.
- If yes, skip populating `ctx.StorageAccessKey` / `ctx.StorageSecretKey` from the OBC secret; instead, set a new `ctx.StorageSTSEnabled = true` flag and store the CCO secret reference.

In `pkg/kustomize/secrets.go` (`FieldGroupFor` for `ComponentObjectStorage`):
- When `ctx.StorageSTSEnabled` is true, omit `AccessKey` / `SecretKey` from the storage config and instead configure Quay's storage driver to use the ambient IAM role via the standard boto credential chain (i.e., leave keys blank so boto falls through to the instance metadata / Web Identity Token file).

In `controllers/quay/quayregistry_controller.go` (main reconcile loop):
- Watch for the CCO `CredentialRequest` reaching `Provisioned` status.
- If not yet provisioned after a configurable timeout: set a `Degraded` condition on `QuayRegistry` with reason `CredentialRequestNotProvisioned` and a message guiding the user to verify the role ARN and OIDC provider.

#### 4. CSV and RBAC Changes

**CSV annotation** (`bundle/manifests/quay-operator.clusterserviceversion.yaml`):

```yaml
features.operators.openshift.io/token-auth-aws: "true"   # changed from "false"
```

**New RBAC rules** (added to ClusterServiceVersion `installModes` / `clusterPermissions`):

```yaml
- apiGroups: ["cloudcredential.openshift.io"]
  resources: ["credentialrequests"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: ["config.openshift.io"]
  resources: ["infrastructures"]
  verbs: ["get", "list", "watch"]
```

The `config.openshift.io/infrastructures` permission is needed to detect whether the cluster is running on AWS and whether the OIDC issuer URL is configured (STS-enabled cluster detection).

#### 5. Cluster Capability Detection

On reconcile, the operator reads the cluster `Infrastructure` CR to detect:
- `platform.type == "AWS"` — skip STS logic on non-AWS clusters
- `status.platformStatus.aws.resourceTags` or OIDC issuer presence — confirm STS is enabled

If the cluster is AWS but does not appear to be an STS cluster (no OIDC issuer), the operator logs a warning and falls back to static credentials even if the annotation is present.

#### 6. RHEL-Based Quay Deployments

For Quay running on bare-metal or VMs (not on OCP), the CCO flow is unavailable. The operator documentation and release notes will describe the manual equivalent:

1. Create an IAM role with the required S3 permissions.
2. Configure the EC2 instance profile or the service account token file path (`AWS_WEB_IDENTITY_TOKEN_FILE`).
3. Set `AWS_ROLE_ARN` in the Quay container environment so boto's `assume_role()` chain picks it up automatically without static keys in `config.yaml`.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| CCO not available on the cluster (OCP < 4.14 or non-OCP) | Operator detects absence of `CredentialRequest` CRD at startup and skips the STS path entirely |
| Role ARN provided but CCO cannot reconcile (wrong ARN, missing OIDC provider) | Operator sets `Degraded` condition with actionable message; does not crash or deadlock |
| Regression: existing static-credential installations broken by upgrade | STS path is only activated by the opt-in annotation; no annotation = no change in behavior |
| Temporary STS credentials expire mid-operation | CCO handles rotation before expiry; Quay's boto client automatically reloads credentials from the credentials file on the next call |
| Bucket-scoped IAM policy requires bucket name at `CredentialRequest` creation time | Bucket name is available from the ObjectBucketClaim before the `CredentialRequest` is created; no ordering issue |

## Design Details

### Feature Flag

No new Quay application-level feature flag is required. The feature is opt-in at the operator level via the annotation. The operator adds no new `QuayRegistry` spec fields in this iteration; the annotation approach matches the convention used by other CCO-integrated operators.

### Graduation Criteria

#### Dev Preview

- Operator creates and manages the `CredentialRequest` on annotated `QuayRegistry` resources.
- Quay successfully authenticates to S3 with STS credentials on a ROSA cluster.
- Graceful fallback and `Degraded` condition are implemented and tested.

#### Tech Preview

- E2E tests pass on OCP 4.14+ with STS-enabled clusters.
- IAM permission documentation is published and reviewed by security team.
- CSV annotation updated to `token-auth-aws: "true"` and verified with OperatorHub metadata tooling.

#### GA

- Feature is enabled by default for all new `QuayRegistry` installations on annotated clusters.
- Upgrade path from static credentials to STS is documented and tested.
- RHEL-based deployment instructions are part of the official Quay documentation.

### Test Plan

- **Unit tests**: Verify `CredentialRequest` is created with correct spec when the ARN annotation is present; verify it is not created when annotation is absent.
- **Unit tests**: Verify the storage config in `config.yaml` omits `AccessKey`/`SecretKey` when `StorageSTSEnabled` is true.
- **Unit tests**: Verify `Degraded` condition is set when `CredentialRequest` is not provisioned within timeout.
- **Integration tests**: On a simulated STS cluster (with CCO mock), verify the full reconcile loop produces a correctly configured Quay instance.
- **E2E tests** (kuttl): On a live ROSA or OCP 4.14+ STS cluster, verify image push and pull succeed with no static AWS keys in any Secret.
- **Regression tests**: On a standard OCP cluster without the annotation, verify behavior is identical to pre-enhancement.

### Upgrade / Downgrade Strategy

- **Upgrade**: Existing installations without the annotation are unaffected. Administrators wishing to migrate to STS must add the annotation post-upgrade and ensure the IAM role exists; the operator will then create the `CredentialRequest` on the next reconcile cycle.
- **Downgrade**: Removing the annotation causes the operator to delete the `CredentialRequest` and revert to static credential sourcing from the ObjectBucketClaim Secret on the next reconcile.

### Version Skew Strategy

The `CredentialRequest` CRD is provided by CCO, which ships as part of OCP. The operator declares a minimum OCP version of 4.14 for this feature. On older clusters the operator must detect the absence of the CRD (via API discovery) and skip the STS path rather than crashing.

## Implementation History

- 2023-07-19 PROJQUAY-5850 filed, feasibility investigation completed.

## Drawbacks

- Adds a dependency on CCO being present and functional for the STS path, which is an additional failure mode not present with static credentials.
- The annotation-based input for the role ARN is unconventional compared to spec fields, but matches the interim pattern used by other OLM operators pending a standardized OLM API for this purpose.

## Alternatives

- **Mount the IAM role directly via a `ServiceAccount` annotation**: Requires the administrator to annotate the Quay app service account with the role ARN and manage IRSA manually. This was the approach before CCO standardization; it is harder for users and does not benefit from CCO's credential rotation.
- **Static credential passthrough**: Continue using `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` from the ObjectBucketClaim Secret. This is the current behavior and remains available as the non-STS fallback, but does not meet ROSA/OSD security requirements.

## Infrastructure Needed

- A ROSA or OCP 4.14+ STS-enabled cluster for E2E testing in CI.
- An IAM role with the documented S3 permissions, and the cluster's OIDC provider, must be provisioned as part of the CI test setup.
