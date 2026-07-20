---
title: Standardized STS Configuration via OLM and CCO for Quay on OpenShift
authors:
  - "@dmesser"
  - "@doconnor"
  - "@tlwu2013"
reviewers:
  - "@jbpratt"
  - "@tlwu2013"
approvers:
  - TBD
creation-date: 2023-07-19
last-updated: 2026-05-26
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

This enhancement integrates the Quay operator with the CCO `CredentialRequest` flow so that Quay application pods on STS-enabled OpenShift clusters (ROSA, OSD) can authenticate to real AWS S3 without static credentials. The implementation follows the pattern defined in OCPSTRAT-171 / OCPSTRAT-6, giving administrators the same STS-enabled installation experience they have with other OLM operators that support token-based authentication.

**Scope**: This enhancement applies exclusively to `ObjectStorage: managed: false` configurations where the customer supplies a real AWS S3 bucket. When `ObjectStorage: managed: true`, the operator provisions a NooBaa/ODF `ObjectBucketClaim` whose credentials are NooBaa-internal and not subject to AWS IAM or STS — that case is unaffected.

## Motivation

When `ObjectStorage` is set to `managed: false`, the customer provides their own AWS S3 configuration in the `configBundleSecret`. Today, those credentials must be static `aws_access_key_id` / `aws_secret_access_key` values. This is incompatible with ROSA and OSD clusters that enforce IAM-role-only policies and prohibit static IAM user keys.

**Why managed ObjectStorage is unaffected**: When `ObjectStorage: managed: true`, the Quay operator creates an `ObjectBucketClaim` (OBC) against the NooBaa/ODF storage class. NooBaa generates its own S3-compatible credentials for the provisioned bucket — these are internal to NooBaa and stored as Kubernetes Secrets with `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` key names for API compatibility, but they authenticate against the NooBaa S3 gateway (an in-cluster service endpoint), not against AWS IAM. There is no AWS STS interaction in this path.

**The gap**: A customer on ROSA who wants to use a real AWS S3 bucket for Quay (instead of NooBaa) must set `ObjectStorage: managed: false`. On ROSA, their security policy prohibits the static IAM keys that the current operator configuration requires. There is no supported path today.

Red Hat's platform strategy (OCPSTRAT-6) mandates that all OLM-managed operators capable of integrating with cloud-provider APIs adopt the CCO-based `CredentialRequest` flow. Quay has been identified as a target operator. A strategic customer (Elevance Health/Anthem) has this as a hard requirement for migrating Quay to ROSA.

### Why the Operator Manages Credentials for Unmanaged Storage

When `ObjectStorage: managed: false`, the operator does not create or manage the S3 bucket — the customer does. So why should the operator manage cloud credentials?

The answer is that the operator is not managing *IAM credentials* in the traditional sense (it never creates IAM users, keys, or policies). Instead, it acts as a **configuration broker** between the OCP platform and the Quay application pods. Specifically:

1. **The customer creates an IAM role** with the necessary S3 permissions and trust policy. This is the customer's responsibility, just like creating the bucket.
2. **The customer supplies the role ARN** via the OLM Subscription (`ROLEARN`), following the standardized OCPSTRAT-171 pattern for OLM operators that support STS.
3. **The operator creates a `CredentialRequest`** — a declarative request that tells CCO "the `quay-app` service account needs to assume this role." The operator does not generate, store, or rotate any credentials itself.
4. **CCO provisions a credentials file** (containing the role ARN and the OIDC token path) and stores it in a Secret. This file is a *configuration pointer*, not a credential — it tells boto "call `AssumeRoleWithWebIdentity` with this role using the projected OIDC token."
5. **The operator mounts this Secret** into the Quay application pods and sets `AWS_SHARED_CREDENTIALS_FILE`. This is the same kind of configuration injection the operator already performs (TLS certificates, config bundles, etc.).

The operator already manages the `quay-app` Deployment, ServiceAccount, and volumes. Adding one more volume mount for CCO-provisioned config is consistent with its existing role. The alternative — asking customers to manually create Secrets, add volume mounts, and set environment variables — would break the standardized OCPSTRAT-171 UX that other operators provide and that OperatorHub surfaces.

### Goals

- Implement the standardized CCO `CredentialRequest` flow for the `quay-app` service account when `ObjectStorage: managed: false` and the cluster is STS-capable.
- Enable Quay application pods to authenticate to AWS S3 using short-lived `AssumeRoleWithWebIdentity` credentials. No static AWS credentials appear in any Kubernetes Secret or in `config.yaml`.
- Follow the standard OLM role ARN injection pattern: the administrator provides the IAM role ARN in the Subscription's `spec.config.env` as `ROLEARN`; OLM injects it into the operator Deployment, and the operator uses it during reconciliation.
- Gracefully fall back to the existing static-credential path when `ROLEARN` is not set or the cluster is not STS-capable.
- Block rollout of the `QuayRegistry` with an informative `RolloutBlocked` condition when `ROLEARN` is set but CCO fails to provision the `CredentialRequest`.
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
   OLM injects ROLEARN into the operator Deployment only (not into quay-app pods)

2. Operator reads ROLEARN from its own environment, detects STS-capable cluster, creates CredentialRequest
   with serviceAccountNames: [quay-app], stsIAMRoleARN: <ROLEARN value>,
   and cloudTokenPath: /var/run/secrets/openshift/serviceaccount/token
   ↓
   CCO validates the operator's OIDC token at cloudTokenPath,
   then provisions a Secret containing a credentials file:

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

As a ROSA administrator, if I provide `ROLEARN` but CCO cannot provision the `CredentialRequest` (wrong ARN, OIDC provider not configured, OCP < 4.14), I want the `QuayRegistry` to report a `RolloutBlocked` condition with a message telling me exactly what to check.

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

OLM's `spec.config.env` injects `ROLEARN` as an environment variable into the **operator Deployment only** — not into workloads the operator itself creates. The quay-app pods are created by the operator's reconcile loop, not by OLM, so `ROLEARN` is not automatically present in them. This is fine: the operator reads `os.Getenv("ROLEARN")` during reconciliation to decide whether to create the `CredentialRequest` and how to configure the quay-app Deployment. The quay-app pods receive their credentials via the mounted CCO Secret and `AWS_SHARED_CREDENTIALS_FILE`, not via `ROLEARN`. If `ROLEARN` is empty, the STS path is skipped entirely.

#### 2. STS-Capable Cluster Detection

Before creating a `CredentialRequest`, the operator confirms the cluster is STS-capable. On each reconcile, in order:

1. **`ROLEARN` present**: If `os.Getenv("ROLEARN")` is empty, skip STS — no further checks needed.
2. **ObjectStorage unmanaged**: If `ComponentObjectStorage` is `managed: true`, skip STS (NooBaa path needs no STS). Log a warning if `ROLEARN` is set with managed storage to alert the admin.
3. **Platform type**: Read `config.openshift.io/v1 Infrastructure cluster`; confirm `status.platformStatus.type == "AWS"`.
4. **CCO mode**: Read `operator.openshift.io/v1 CloudCredential cluster`; confirm `spec.credentialsMode` is not `Mint` or `Passthrough` (those modes produce static keys, not web-identity config). Empty `credentialsMode` on AWS means STS mode.
5. **CRD availability**: Confirm `credentialsrequests.cloudcredential.openshift.io` CRD exists via API discovery. Absent on OCP < 4.14 or non-OCP environments.

New RBAC rules required in CSV as **`clusterPermissions`** (not namespace-scoped `permissions`):

`config.openshift.io/infrastructures` and `operator.openshift.io/cloudcredentials` are cluster-scoped resources (singleton objects named `cluster`). They cannot be accessed via namespace-scoped RBAC. Currently, the quay-operator CSV uses only namespace-scoped `permissions` — adding `clusterPermissions` means OLM will create a ClusterRole and ClusterRoleBinding for the operator ServiceAccount. This is a material change to the operator's OLM security footprint and should be reviewed by the security team.

For reference, the OADP operator's CSV already uses `clusterPermissions` with `cloudcredential.openshift.io/credentialsrequests` RBAC, confirming this is the correct approach for CCO integration.

```yaml
# CSV spec.install.spec.clusterPermissions (new section)
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

The operator creates the `CredentialRequest` programmatically during reconciliation (not packaged in the bundle — OKD documentation explicitly states that bundled CredentialRequests are not supported). One `CredentialRequest` is created per `QuayRegistry` in the registry's namespace, owned by the `QuayRegistry` for garbage collection.

**Namespace**: The CCO README states *"CredentialsRequests should be created in the openshift-cloud-credential-operator namespace"* — this instruction targets CVO-managed (core platform) operators that ship CredentialRequests as static manifests in the release image. CCO watches CredentialRequests across all namespaces (confirmed via CCO source: the controller registers a cluster-wide watch with no namespace filtering). For OLM-managed operators, creating the CR in the operator's namespace enables `ownerReference`-based garbage collection and follows the pattern used by the EFS CSI driver:

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
  cloudTokenPath: /var/run/secrets/openshift/serviceaccount/token
```

`stsIAMRoleARN` (available since OCP 4.14 CCO) tells CCO to produce a web-identity credentials file rather than static IAM user keys. `serviceAccountNames: [quay-app]` is a required enforcement field — CCO rejects CredentialRequests without it.

`cloudTokenPath` must be set explicitly to `/var/run/secrets/openshift/serviceaccount/token`. CCO's default (`/var/run/secrets/kubernetes.io/serviceaccount/token`) is the standard Kubernetes service account token, which has the wrong audience for OpenShift STS. The projected token at the OpenShift-specific path carries `audience: openshift`, which the cluster's OIDC provider expects when validating `AssumeRoleWithWebIdentity` calls.

The `statementEntries` document the IAM permissions Quay requires. In STS mode, CCO does not enforce these — the actual permissions come from the customer's IAM role policy attached to the role ARN. `statementEntries` are used by `ccoctl` in Mint mode to generate IAM policies and are included here for documentation and auditability. The `resource: "*"` is used because the operator does not know the customer's bucket name when storage is unmanaged; the actual bucket-scoped IAM policy is the customer's responsibility when creating the role.

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

**Credential conflict detection**: If a customer's existing `configBundleSecret` contains `s3_access_key` / `s3_secret_key` in `DISTRIBUTED_STORAGE_CONFIG`, boto3 will use those static credentials **instead of** the CCO-provisioned credentials file (explicit credentials take priority in boto's credential chain). The STS pathway would silently do nothing.

When `ROLEARN` is set, the operator must inspect the customer's `configBundleSecret` for static AWS credentials in the storage configuration. If both are present, the operator should:
1. Set a `RolloutBlocked` condition with reason `ConflictingCredentials` and a message instructing the customer to remove `s3_access_key`/`s3_secret_key` from their `configBundleSecret`.
2. Not roll out the Quay pods until the conflict is resolved.

This prevents a confusing state where STS appears configured but static credentials silently take precedence.

#### 6. RolloutBlocked Condition

The quay-operator uses `Available`, `RolloutBlocked`, and `ComponentsCreated` as condition types on `QuayRegistry` status. There is no `RolloutBlocked` condition type defined.

If the `CredentialRequest` has not reached `status.provisioned == true` within a configurable timeout (default: 5 minutes) after `ROLEARN` is detected, the operator sets:

```
type:    RolloutBlocked
status:  True
reason:  CredentialRequestNotProvisioned
message: "CCO has not provisioned CredentialRequest <name>. Verify: (1) the IAM role ARN
          in ROLEARN is correct, (2) the cluster OIDC provider is configured, (3) CCO is
          not in Mint or Passthrough mode. See <docs link>."
```

This prevents the operator from rolling out Quay pods until the `CredentialRequest` is provisioned, and makes the `QuayRegistry` status clearly indicate why the rollout is blocked. The existing status controller aggregation logic for `RolloutBlocked` applies — `Available` will remain `False` while any `RolloutBlocked` condition is `True`.

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

Three additions are required in the ClusterServiceVersion:

#### Annotation

```yaml
features.operators.openshift.io/token-auth-aws: "true"   # changed from "false"
```

This annotation is a declarative signal to OperatorHub's web console. When set to `"true"`, the console displays a role ARN input field during operator installation, prompting the administrator to provide their IAM role ARN. The annotation does not automatically inject `ROLEARN` — the administrator must supply the value, which OLM then propagates to the operator Deployment via `Subscription.spec.config.env`.

#### Projected OIDC Token Volume for the Operator Pod

The operator pod needs a projected `bound-sa-token` volume so that its own OIDC token is available at the path specified by `cloudTokenPath` in the `CredentialRequest`. Without this volume, the directory `/var/run/secrets/openshift/serviceaccount/` does not exist in the operator pod and CCO cannot validate the `CredentialRequest`.

Add to the operator Deployment spec in the CSV:

```yaml
spec:
  template:
    spec:
      containers:
        - name: quay-operator
          volumeMounts:
            - name: bound-sa-token
              mountPath: /var/run/secrets/openshift/serviceaccount
              readOnly: true
      volumes:
        - name: bound-sa-token
          projected:
            sources:
              - serviceAccountToken:
                  path: token
                  audience: openshift
```

#### Additional RBAC (clusterPermissions)

The operator needs read access to cluster-scoped infrastructure and cloud credential resources for STS-capability detection. These must be `clusterPermissions` in the CSV (not namespace-scoped `permissions`), since `infrastructures` and `cloudcredentials` are cluster-scoped singletons. See Section 2 for the full RBAC list including `credentialsrequests`.

```yaml
# CSV spec.install.spec.clusterPermissions (new section)
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
| IAM role ARN wrong or trust policy misconfigured | `RolloutBlocked` condition with actionable message; operator retries each reconcile |
| CredentialRequest rejected by CCO (missing `serviceAccountNames`) | CCO 4.14+ requires this field; operator always populates it |
| Regression on non-STS upgrades | STS path requires `ROLEARN` env var; existing Subscriptions without it are fully unaffected |
| Static credentials in `configBundleSecret` coexist with `ROLEARN` | Operator detects conflict and blocks rollout with `ConflictingCredentials` reason; admin must remove static keys |
| Multipart upload in-flight when OIDC token rotates | boto re-fetches the token file on each credential refresh cycle; the token at the path is updated by kubelet before expiry |

### Design Decisions

#### Credential Delivery: Runtime CredentialRequest (Approach A)

**Decision**: The operator creates `CredentialRequest` objects at runtime during reconciliation. CCO provisions the corresponding Secret. This is the officially documented pattern for OLM-managed operators.

**Rationale**:

1. The OKD/OCP Operator SDK documentation explicitly prescribes runtime CredentialRequest creation for OLM operators and states: *"Adding a CredentialsRequest object to the Operator bundle is not currently supported."*
2. The EFS CSI driver ([openshift/csi-operator PR #251](https://github.com/openshift/csi-operator/pull/251), merged 2024-08-06) implements this exact pattern: a `stsCredentialsRequestHook` reads `os.Getenv("ROLEARN")` and injects it into the CredentialRequest's `stsIAMRoleARN` field via `unstructured.SetNestedField`.
3. This approach delegates credential file format and lifecycle to CCO, avoiding tight coupling to the AWS credentials file format.
4. CCO 4.14+ detects STS-enabled clusters (via `IsTimedTokenCluster()`) even in Manual `credentialsMode` and semi-automates Secret provisioning for runtime CredentialRequests.

#### Considered Alternatives

| Approach | How it works | Why not selected |
|---|---|---|
| **B. Bundle-shipped CredentialRequest** | `CredentialRequest` manifest shipped in bundle under `manifests/`; `ccoctl` or CCO processes it at install time | Explicitly unsupported by OLM for operator bundles. Cannot support per-registry ARNs (static manifest). |
| **C. Direct Secret creation** | Operator reads `ROLEARN`, creates the AWS credentials Secret directly (role ARN + token path), bypassing CCO entirely | Deviates from official OLM guidance. Couples operator to AWS credential file format. OADP uses this approach (`pkg/credentials/stsflow/stsflow.go`) but bypasses CCO entirely — the RBAC for `credentialsrequests` in OADP's CSV exists because OLM requires it when `token-auth-aws: "true"` is set, not because OADP exercises it. |

**Note on OADP**: OADP reads `ROLEARN` from the environment and creates the AWS credential Secret directly, bypassing CCO's `CredentialRequest` flow. While this works in production on ROSA, it is a deviation from the officially documented OLM pattern. cert-manager takes a different approach: it relies on admins to create CredentialRequests externally (via `ccoctl`) and consumes the resulting Secret.

#### Storage Driver: `S3Storage` with Web Identity (not `STSS3Storage`)

**Decision**: Use `S3Storage` for the CCO path. `STSS3Storage` remains available for RHEL-based deployments where cross-account assume-role with static keys is the only option.

The `STSS3Storage` driver in `quay/quay` uses `sts:AssumeRole` with static IAM user keys to obtain temporary credentials. This is a different STS flow from the CCO/web-identity path.

With the CCO approach, the credentials file provisioned by CCO contains `role_arn` and `web_identity_token_file`. When `AWS_SHARED_CREDENTIALS_FILE` is set, boto3's standard credential chain resolves this automatically via `AssumeRoleWithWebIdentity` — no Quay code changes needed. The standard `S3Storage` driver works as-is because boto handles credential resolution transparently.

## Design Details

### Graduation Criteria

#### Dev Preview

- Operator reads `ROLEARN`, detects STS-capable cluster, creates `CredentialRequest` for `quay-app`.
- CCO-provisioned credentials file is mounted into Quay app pods; `AWS_SHARED_CREDENTIALS_FILE` is set.
- Image push and pull succeed on a ROSA cluster with `ObjectStorage: managed: false` and no static AWS credentials anywhere.
- Graceful fallback when `ROLEARN` is absent.
- `RolloutBlocked` condition when `CredentialRequest` not provisioned.

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
- **Unit**: `CredentialRequest.status.provisioned == false` past timeout → `RolloutBlocked` condition set.
- **Unit**: CCO in Mint mode → STS path skipped.
- **Unit**: `ROLEARN` set + `configBundleSecret` contains `s3_access_key`/`s3_secret_key` → `RolloutBlocked` with `ConflictingCredentials` reason.
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
- 2026-04-16 Targeted for Quay 3.19 (Q4 2026); GCS WIF (PROJQUAY-7729) to share operator-side pattern.
- 2026-04-17 Initial enhancement PR opened; review feedback on authors, credential brokering rationale.
- 2026-04-28 Review feedback: `cloudTokenPath` and `bound-sa-token` volume additions required.
- 2026-05-21 Enhancement updated to address review feedback; open design questions documented.
- 2026-05-26 Addressed bcaton85 review: corrected OLM env propagation, OADP precedent, condition type (RolloutBlocked), clusterPermissions, credential conflict detection.
- 2026-06-02 Resolved open design questions after deep research into CCO source code and OLM operator precedent (EFS CSI driver, OADP). Committed to Approach A (runtime CredentialRequest). Added cloudTokenPath default clarification, statementEntries STS-mode behavior, token-auth-aws annotation explanation, CredentialRequest namespace guidance for OLM operators.

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
