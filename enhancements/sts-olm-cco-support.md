---
title: Standardized STS Configuration via OLM and CCO for Quay on OpenShift
authors:
  - TBD
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

## Summary

AWS STS (Security Token Service) based authentication eliminates the need for static, long-lived AWS access keys by exchanging a Kubernetes-projected OIDC service account token for short-lived IAM credentials via `sts:AssumeRoleWithWebIdentity`. OpenShift's Cloud Credential Operator (CCO) standardizes this across OLM-managed operators through the `CredentialRequest` API.

This enhancement integrates the Quay operator with the CCO `CredentialRequest` flow so that Quay application pods on STS-enabled OpenShift clusters (ROSA, OSD) can authenticate to real AWS S3 without static credentials. The implementation follows the pattern defined in OCPSTRAT-171 / OCPSTRAT-6, giving administrators the same installation experience they have with other CCO-integrated OLM operators such as OADP and cert-manager.

**Scope**: This enhancement applies exclusively to `ObjectStorage: managed: false` configurations where the customer supplies a real AWS S3 bucket. When `ObjectStorage: managed: true`, the operator provisions a NooBaa/ODF `ObjectBucketClaim` whose credentials are NooBaa-internal and not subject to AWS IAM or STS — that case is unaffected.

## Motivation

When `ObjectStorage` is set to `managed: false`, the customer provides their own AWS S3 configuration in the `configBundleSecret`. Today, those credentials must be static `aws_access_key_id` / `aws_secret_access_key` values. This is incompatible with ROSA and OSD clusters that enforce IAM-role-only policies and prohibit static IAM user keys.

**Why managed ObjectStorage is unaffected**: When `ObjectStorage: managed: true`, the Quay operator creates an `ObjectBucketClaim` (OBC) against the NooBaa/ODF storage class. NooBaa generates its own S3-compatible credentials for the provisioned bucket — these are internal to NooBaa and stored as Kubernetes Secrets with `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` key names for API compatibility, but they authenticate against the NooBaa S3 gateway (an in-cluster service endpoint), not against AWS IAM. There is no AWS STS interaction in this path.

**The gap**: A customer on ROSA who wants to use a real AWS S3 bucket for Quay (instead of NooBaa) must set `ObjectStorage: managed: false`. On ROSA, their security policy prohibits the static IAM keys that the current operator configuration requires. There is no supported path today.

Red Hat's platform strategy (OCPSTRAT-6) mandates that all OLM-managed operators capable of integrating with cloud-provider APIs adopt the CCO-based `CredentialRequest` flow. Quay has been identified as a target operator. A strategic customer (Elevance Health/Anthem) has this as a hard requirement for migrating Quay to ROSA.

### Goals

- Implement the standardized CCO `CredentialRequest` flow for the `quay-app` service account when `ObjectStorage: managed: false` and the cluster is STS-capable.
- Enable Quay application pods to authenticate to AWS S3 using short-lived `AssumeRoleWithWebIdentity` credentials. No static AWS credentials appear in any Kubernetes Secret or in `config.yaml`.
- Follow the standard OLM role ARN injection pattern: the administrator provides the IAM role ARN in the Subscription's `spec.config.env` as `ROLEARN`; OLM propagates it to all operator-managed pods.
- Gracefully fall back to the existing static-credential path when `ROLEARN` is not set or the cluster is not STS-capable.
- Degrade the `QuayRegistry` with an informative condition when `ROLEARN` is set but CCO fails to provision the `CredentialRequest`.
- Document the required IAM permissions and IAM role trust policy.
- Annotate the Quay CSV with `features.operators.openshift.io/token-auth-aws: "true"`.

### Non-Goals

- STS for `ObjectStorage: managed: true` (NooBaa/ODF). NooBaa manages its own backing-store credentials independently.
- Support for OCP versions older than 4.14.
- STS for non-AWS cloud providers (Azure Workload Identity, GCP WIF tracked in PROJQUAY-7729).
- Automatic IAM role, IAM policy, or OIDC provider creation in the customer's AWS account.
- Changes to the Quay application (`quay/quay`); all changes are confined to the operator.

## Proposal

### How the Credential Flow Works

Understanding the credential flow is essential because this is NOT the traditional CCO "Mint" mode that produces long-lived IAM user keys. In STS/OIDC mode, CCO acts as a configuration broker, not a key dispenser.

```
1. Admin installs operator via Subscription with spec.config.env: [{name: ROLEARN, value: <arn>}]
   ↓
   OLM injects ROLEARN into all pods managed by this operator (including quay-app pods)

2. Operator reads ROLEARN, detects STS-capable cluster, creates CredentialRequest
   with serviceAccountNames: [quay-app] and stsIAMRoleARN: <ROLEARN value>
   ↓
   CCO provisions a Secret containing a credentials file:

     [default]
     sts_regional_endpoints = regional
     role_arn = arn:aws:iam::123456789012:role/quay-s3-role
     web_identity_token_file = /var/run/secrets/openshift/serviceaccount/token

3. Operator mounts this Secret into quay-app pods as a volume
   and sets AWS_SHARED_CREDENTIALS_FILE=/var/run/secrets/cloud/credentials
   ↓
   OCP automatically projects a signed OIDC token for the quay-app service account
   at /var/run/secrets/openshift/serviceaccount/token in every quay-app pod

4. When boto3 in a quay-app pod makes an S3 API call:
   - boto reads AWS_SHARED_CREDENTIALS_FILE
   - Sees role_arn + web_identity_token_file → calls sts:AssumeRoleWithWebIdentity
   - AWS validates the OIDC token against the cluster's OIDC provider endpoint
   - AWS confirms the token subject matches system:serviceaccount:NAMESPACE:quay-app
     (as constrained by the IAM role trust policy)
   - AWS returns temporary AccessKeyId/SecretAccessKey/SessionToken
   - boto caches these and refreshes transparently before they expire

   No static credentials appear anywhere. No operator reconcile loop is needed for rotation.
```

**Why the CredentialRequest targets `quay-app` and not the operator SA**: The `serviceAccountNames` field in the `CredentialRequest` is a required security field (enforced by CCO since OCP 4.14 — CredentialRequests without it are rejected). It tells CCO which Kubernetes service accounts are authorized to use the provisioned cloud credential. Since it is the `quay-app` pods that call S3 — not the operator pod — the CredentialRequest must reference `quay-app`. The operator acts as a credential broker: it creates the CredentialRequest on behalf of the application it manages, then mounts the resulting Secret into those application pods.

### User Stories

#### Story 1 — ROSA administrator installs Quay with real AWS S3

As a ROSA cluster administrator whose security policy prohibits static IAM keys, I want to install Quay via OperatorHub, supply my pre-created IAM role ARN once (in the Subscription), configure Quay with my S3 bucket details but no credentials, and have the operator automatically wire up STS authentication for the Quay application pods.

#### Story 2 — Existing Quay installation with static credentials is unaffected

As an OCP administrator running Quay today (either with NooBaa managed storage or with unmanaged S3 using static keys), I want to upgrade the Quay operator and have zero behavior change — no `ROLEARN` is set in my Subscription, so the operator continues to use the credentials I have already provided.

#### Story 3 — Incomplete STS configuration is surfaced clearly

As a ROSA administrator, if I provide `ROLEARN` but CCO cannot provision the `CredentialRequest` (wrong ARN, OIDC provider not configured, OCP < 4.14), I want the `QuayRegistry` to report a `Degraded` condition with a message telling me exactly what to check.

### Implementation Details

#### 1. Role ARN Input — OLM Subscription

The administrator provides the IAM role ARN via the Subscription, following the OCPSTRAT-171 standard:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: quay-operator
  namespace: openshift-operators
spec:
  channel: stable-3.13
  name: quay-operator
  config:
    env:
      - name: ROLEARN
        value: "arn:aws:iam::123456789012:role/quay-s3-role"
```

OLM propagates `ROLEARN` as an environment variable into all Deployments managed by the operator — both the operator pod and the Quay application pods. The operator reads `os.Getenv("ROLEARN")` during reconciliation. If it is empty, the STS path is skipped entirely.

#### 2. STS-Capable Cluster Detection

Before creating a `CredentialRequest`, the operator confirms the cluster is STS-capable. On each reconcile, in order:

1. **`ROLEARN` present**: If `os.Getenv("ROLEARN")` is empty, skip STS — no further checks needed.
2. **ObjectStorage unmanaged**: If `ComponentObjectStorage` is `managed: true`, skip STS (NooBaa path needs no STS). Log a warning if `ROLEARN` is set with managed storage to alert the admin.
3. **Platform type**: Read `config.openshift.io/v1 Infrastructure cluster`; confirm `status.platformStatus.type == "AWS"`.
4. **CCO mode**: Read `operator.openshift.io/v1 CloudCredential cluster`; confirm `spec.credentialsMode` is not `Mint` or `Passthrough` (those modes produce static keys, not web-identity config). Empty `credentialsMode` on AWS means STS mode.
5. **CRD availability**: Confirm `credentialsrequests.cloudcredential.openshift.io` CRD exists via API discovery. Absent on OCP < 4.14 or non-OCP environments.

New RBAC rules required in CSV:

```yaml
- apiGroups: ["config.openshift.io"]
  resources: ["infrastructures"]
  verbs: ["get"]
- apiGroups: ["operator.openshift.io"]
  resources: ["cloudcredentials"]
  verbs: ["get"]
- apiGroups: ["cloudcredential.openshift.io"]
  resources: ["credentialsrequests"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
```

#### 3. CredentialRequest — Created at Runtime

The operator creates the `CredentialRequest` programmatically during reconciliation (not packaged in the bundle — OKD documentation explicitly states that bundled CredentialRequests are not supported). One `CredentialRequest` is created per `QuayRegistry` in the registry's namespace, owned by the `QuayRegistry` for garbage collection:

```yaml
apiVersion: cloudcredential.openshift.io/v1
kind: CredentialRequest
metadata:
  name: <quayregistry-name>-quay-app
  namespace: <quayregistry-namespace>
  ownerReferences:
    - apiVersion: quay.redhat.com/v1
      kind: QuayRegistry
      name: <quayregistry-name>
spec:
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: AWSProviderSpec
    stsIAMRoleARN: "<value of ROLEARN env var>"
    statementEntries:
      - effect: Allow
        action:
          - s3:GetObject
          - s3:PutObject
          - s3:DeleteObject
          - s3:HeadObject
          - s3:CreateMultipartUpload
          - s3:UploadPart
          - s3:CompleteMultipartUpload
          - s3:AbortMultipartUpload
          - s3:ListBucketMultipartUploads
        resource: "*"
      - effect: Allow
        action:
          - s3:ListBucket
          - s3:HeadBucket
          - s3:GetBucketLocation
          - s3:GetBucketCors
          - s3:PutBucketCors
        resource: "*"
  secretRef:
    name: <quayregistry-name>-quay-app-aws
    namespace: <quayregistry-namespace>
  serviceAccountNames:
    - quay-app
```

`stsIAMRoleARN` (available since OCP 4.14 CCO) tells CCO to produce a web-identity credentials file rather than static IAM user keys. `serviceAccountNames: [quay-app]` is a required enforcement field — CCO rejects CredentialRequests without it. The `statementEntries` use `resource: "*"` because the operator does not know the customer's bucket name when storage is unmanaged; the actual bucket-scoped IAM policy is the customer's responsibility when creating the role.

CCO produces the Secret `<quayregistry-name>-quay-app-aws` containing:

```ini
[default]
sts_regional_endpoints = regional
role_arn = arn:aws:iam::123456789012:role/quay-s3-role
web_identity_token_file = /var/run/secrets/openshift/serviceaccount/token
```

#### 4. Mounting Credentials into Quay Application Pods

Once `CredentialRequest.status.provisioned == true`, the operator adds to the Quay app Deployment:

- A `volume` sourced from the CCO Secret (`<quayregistry-name>-quay-app-aws`)
- A `volumeMount` at `/var/run/secrets/cloud/` in the Quay container
- An env var `AWS_SHARED_CREDENTIALS_FILE=/var/run/secrets/cloud/credentials`

OCP automatically projects a fresh OIDC-signed token for the `quay-app` service account at `/var/run/secrets/openshift/serviceaccount/token` — this is standard OCP behavior for pods on STS-enabled clusters and requires no additional volume configuration.

#### 5. Quay `config.yaml` Changes

In `pkg/kustomize/secrets.go`, when `ctx.StorageSTSEnabled` is true, the storage configuration omits all credential fields:

```yaml
DISTRIBUTED_STORAGE_CONFIG:
  default:
    - S3Storage
    - host: s3.amazonaws.com
      s3_bucket: <customer-provided>
      s3_region: <customer-provided>
      storage_path: /datastorage/registry
      # No aws_access_key_id or aws_secret_access_key
      # boto resolves via AWS_SHARED_CREDENTIALS_FILE → AssumeRoleWithWebIdentity
```

Note: the storage type becomes `S3Storage` (not `RHOCSStorage`). `RHOCSStorage` is only used for managed NooBaa storage; for real AWS S3 the customer's unmanaged config already specifies the correct storage type.

The operator does not generate the storage configuration for unmanaged storage — that comes from the customer's `configBundleSecret`. The operator only ensures that `AWS_SHARED_CREDENTIALS_FILE` is set on the pods. No modification of the customer's `config.yaml` content is needed or performed.

#### 6. Degraded Condition

If the `CredentialRequest` has not reached `status.provisioned == true` within a configurable timeout (default: 5 minutes) after `ROLEARN` is detected, the operator sets:

```
type:    Degraded
status:  True
reason:  CredentialRequestNotProvisioned
message: "CCO has not provisioned CredentialRequest <name>. Verify: (1) the IAM role ARN
          in ROLEARN is correct, (2) the cluster OIDC provider is configured, (3) CCO is
          not in Mint or Passthrough mode. See <docs link>."
```

The operator does not roll out Quay until the `CredentialRequest` is provisioned.

### Required IAM Permissions

Derived from static analysis of all boto3 call sites in `storage/cloud.py` (`quay/quay`):

**Object-level actions** (resource: `arn:aws:s3:::BUCKET/*`):

| IAM Action | boto3 Call | Purpose |
|---|---|---|
| `s3:GetObject` | `obj.get()` | Download blobs and manifests |
| `s3:PutObject` | `obj.put()` | Upload blobs and manifests |
| `s3:DeleteObject` | `obj.delete()` | Delete blobs during GC |
| `s3:HeadObject` | `head_object()` | Check object existence/size |
| `s3:CreateMultipartUpload` | `initiate_multipart_upload()` | Start chunked layer upload |
| `s3:UploadPart` | `part.upload()` | Upload chunk |
| `s3:CompleteMultipartUpload` | `mp.complete()` | Finalize layer upload |
| `s3:AbortMultipartUpload` | `mp.abort()` | Clean up failed uploads |
| `s3:ListBucketMultipartUploads` | `list_objects` paginator | Find stale multipart uploads |

**Bucket-level actions** (resource: `arn:aws:s3:::BUCKET`):

| IAM Action | boto3 Call | Purpose |
|---|---|---|
| `s3:ListBucket` | `list_objects_v2()` | Enumerate objects for cleanup |
| `s3:HeadBucket` | `head_bucket()` | Verify bucket accessibility |
| `s3:GetBucketLocation` | implicit in presigned URLs | Determine bucket region |
| `s3:GetBucketCors` | `get_bucket_cors()` | Read CORS config |
| `s3:PutBucketCors` | `put_bucket_cors()` | Set CORS for browser uploads |

**Example IAM role policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:HeadObject",
        "s3:CreateMultipartUpload", "s3:UploadPart", "s3:CompleteMultipartUpload",
        "s3:AbortMultipartUpload", "s3:ListBucketMultipartUploads"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket", "s3:HeadBucket", "s3:GetBucketLocation",
        "s3:GetBucketCors", "s3:PutBucketCors"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    }
  ]
}
```

**IAM role trust policy** (must reference the cluster's OIDC provider):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/OIDC_PROVIDER_URL"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "OIDC_PROVIDER_URL:sub": "system:serviceaccount:QUAY_NAMESPACE:quay-app"
        }
      }
    }
  ]
}
```

### CSV Changes

```yaml
features.operators.openshift.io/token-auth-aws: "true"   # changed from "false"
```

### RHEL-Based Quay Deployments

For Quay running outside OCP (bare-metal, VMs), no CCO or OLM is available.

**Option A — EC2 instance profile (recommended):** Attach an IAM instance profile with the permissions above to the EC2 instance. boto uses the instance metadata service (IMDSv2) automatically. No credentials in `config.yaml`.

**Option B — `STSS3Storage` (cross-account assume-role):** Quay ships `STSS3Storage` in `storage/cloud.py` (lines 1235–1284). It calls `sts:AssumeRole` using an IAM user's static keys, then auto-refreshes the temporary credentials. This still requires static IAM user keys (scoped to `sts:AssumeRole` only), so it does not satisfy ROSA's prohibition on all static keys but is an improvement over long-lived S3 keys.

```yaml
DISTRIBUTED_STORAGE_CONFIG:
  default:
    - STSS3Storage
    - sts_role_arn: arn:aws:iam::123456789012:role/quay-s3-role
      sts_user_access_key: AKIAIOSFODNN7EXAMPLE
      sts_user_secret_key: <secret>
      s3_bucket: quay-bucket
      s3_region: us-east-1
      storage_path: /datastorage/registry
```

**Option C — `AWS_ROLE_ARN` + web identity token file:** Set `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` in the Quay container environment and use `S3Storage` with no credentials in `config.yaml`. Requires an externally managed OIDC token file on the host.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| `ROLEARN` set but `ObjectStorage` is managed (NooBaa) | Operator logs a warning and skips STS path; NooBaa credentials continue to be used |
| CCO absent or in Mint/Passthrough mode | Detection step falls back to static credentials; logs the reason |
| IAM role ARN wrong or trust policy misconfigured | `Degraded` condition with actionable message; operator retries each reconcile |
| CredentialRequest rejected by CCO (missing `serviceAccountNames`) | CCO 4.14+ requires this field; operator always populates it |
| Regression on non-STS upgrades | STS path requires `ROLEARN` env var; existing Subscriptions without it are fully unaffected |
| Multipart upload in-flight when OIDC token rotates | boto re-fetches the token file on each credential refresh cycle; the token at the path is updated by kubelet before expiry |

## Design Details

### Graduation Criteria

#### Dev Preview

- Operator reads `ROLEARN`, detects STS-capable cluster, creates `CredentialRequest` for `quay-app`.
- CCO-provisioned credentials file is mounted into Quay app pods; `AWS_SHARED_CREDENTIALS_FILE` is set.
- Image push and pull succeed on a ROSA cluster with `ObjectStorage: managed: false` and no static AWS credentials anywhere.
- Graceful fallback when `ROLEARN` is absent.
- `Degraded` condition when `CredentialRequest` not provisioned.

#### Tech Preview

- E2E kuttl tests pass on OCP 4.14+ STS-enabled clusters in CI.
- IAM policy document reviewed by security team and published in operator documentation.
- CSV annotation `token-auth-aws: "true"` validated with OperatorHub metadata tooling.
- Behavior with managed storage (NooBaa + `ROLEARN` set) is tested and warning is verified.

#### GA

- Upgrade path from static unmanaged S3 credentials to STS is documented and tested.
- RHEL-based deployment guidance published in official Quay documentation.
- Alert or status metric for `CredentialRequestNotProvisioned` available.

### Test Plan

- **Unit**: `ROLEARN` set + unmanaged storage + STS cluster → `CredentialRequest` created with correct `stsIAMRoleARN` and `serviceAccountNames: [quay-app]`.
- **Unit**: `ROLEARN` absent → no `CredentialRequest` created, no behavior change.
- **Unit**: `ROLEARN` set + managed storage → no `CredentialRequest`, warning logged.
- **Unit**: `CredentialRequest.status.provisioned == false` past timeout → `Degraded` condition set.
- **Unit**: CCO in Mint mode → STS path skipped.
- **Integration**: With CCO mock, verify Quay app Deployment has the volume mount and `AWS_SHARED_CREDENTIALS_FILE` env var after `CredentialRequest` is provisioned.
- **E2E (kuttl)**: On live ROSA + unmanaged S3: push and pull images; confirm no AWS credentials in any Secret or `config.yaml`.
- **Regression**: Standard OCP cluster without `ROLEARN`, managed or unmanaged storage — verify identical behavior to pre-enhancement.

### Upgrade / Downgrade Strategy

- **Upgrade + opt-in to STS**: Add `ROLEARN` to the Subscription `spec.config.env` post-upgrade. Operator creates `CredentialRequest` on next reconcile; once provisioned, rolls out Quay pods with the credentials file mount. Static credentials in `configBundleSecret` can be removed after confirming S3 access works.
- **Opt-out / downgrade**: Remove `ROLEARN` from Subscription. Operator deletes the `CredentialRequest` (via ownerRef GC) and removes the volume mount from Quay pods on next reconcile. Customer must restore static credentials to `configBundleSecret`.

### Version Skew Strategy

The `CredentialRequest` CRD is provided by CCO, which ships with OCP. The operator performs API discovery at startup and skips the STS path entirely if the CRD is absent, preventing crashes on OCP < 4.14 or non-OCP clusters.

## Implementation History

- 2023-07-19 PROJQUAY-5850 filed; feasibility investigation completed.

## Drawbacks

- Adds a CCO and OIDC dependency for the STS path. If CCO is unhealthy or the OIDC provider is misconfigured, Quay storage is unavailable until resolved.
- `ROLEARN` is cluster-scoped (set at the Subscription level), so all `QuayRegistry` instances managed by this operator share the same IAM role. Per-registry roles are not supported in this iteration.

## Alternatives

- **Per-`QuayRegistry` annotation for role ARN**: More granular than Subscription-level `ROLEARN`, but deviates from the OCPSTRAT-171 standardized flow. Operators that deviate create inconsistent UX for administrators.
- **`eks.amazonaws.com/role-arn` ServiceAccount annotation only**: Simpler, no CCO dependency, but the operator manages the `quay-app` ServiceAccount and would overwrite manually applied annotations on reconcile. Does not integrate with OperatorHub STS installation UX.
- **`STSS3Storage` (cross-account assume-role)**: Already available in `quay/quay`; works today but still requires static IAM user keys. Does not meet ROSA security requirements.

## Infrastructure Needed

- A ROSA or OCP 4.14+ STS-enabled cluster for E2E testing in CI.
- An AWS IAM role with the permissions above, with the CI cluster's OIDC provider in the trust policy, scoped to `system:serviceaccount:CI_NAMESPACE:quay-app`.
- `ccoctl` tooling in CI for OIDC provider setup during test cluster provisioning.
