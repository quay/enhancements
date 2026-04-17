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

AWS STS (Security Token Service) based authentication eliminates the need for static, long-lived AWS access keys by exchanging a Kubernetes-projected service account token for short-lived IAM credentials via the `AssumeRoleWithWebIdentity` OIDC flow. OpenShift's Cloud Credential Operator (CCO) standardizes this across all OLM-managed operators through the `CredentialRequest` API.

This enhancement integrates the Quay operator with the CCO `CredentialRequest` flow so that Quay on STS-enabled OpenShift clusters (ROSA, OSD) can authenticate to AWS object storage without static credentials. The implementation follows the pattern defined in OCPSTRAT-171 / OCPSTRAT-6, giving administrators the same experience they have with any other CCO-integrated OLM operator.

**Scope**: This enhancement covers `ObjectStorage: managed: true` only. The unmanaged ObjectStorage case is analyzed in the [Unmanaged ObjectStorage and STS](#unmanaged-objectstorage-and-sts) section below.

## Motivation

Quay uses AWS S3 (or S3-compatible object storage via OpenShift Data Foundation/RHOCS) as its primary blob storage backend. Today, the operator configures Quay with static `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` credentials sourced from an ObjectBucketClaim Secret. This approach has several drawbacks:

- **Static credentials** are a security liability: they do not rotate automatically and must be managed out-of-band.
- **ROSA and OSD clusters** commonly enforce IAM-role-only policies and prohibit static IAM user keys entirely, making Quay incompatible with these environments.
- **Inconsistency** across OLM operators: other operators (e.g., cert-manager, OADP) have already adopted the CCO `CredentialRequest` flow. Quay's divergence creates operational friction for administrators familiar with the standard pattern.
- **Customer demand**: Elevance Health (Anthem) and other strategic accounts require STS-based auth as a hard requirement for deploying Quay on ROSA.

Red Hat's platform strategy (OCPSTRAT-6) mandates that all OLM-managed operators capable of integrating with cloud provider APIs adopt the CCO-based `CredentialRequest` flow. Quay has been explicitly identified as a target operator.

### Goals

- Implement the standardized CCO `CredentialRequest` flow in the Quay operator for AWS STS, for managed ObjectStorage.
- Enable Quay application pods to authenticate to AWS S3 using short-lived `AssumeRoleWithWebIdentity` credentials derived from a Kubernetes-projected OIDC service account token. No static credentials appear anywhere in Kubernetes Secrets.
- Gracefully fall back to the existing static-credential path when no IAM role ARN is provided, preserving backwards compatibility.
- Degrade the `QuayRegistry` with an informative condition when a role ARN is configured but CCO fails to provision the `CredentialRequest`.
- Document the exact IAM permissions required by Quay and provide instructions for creating the IAM role.
- Provide guidance for RHEL-based (non-OCP) deployments.
- Annotate the Quay CSV with `features.operators.openshift.io/token-auth-aws: "true"`.

### Non-Goals

- Support for OCP versions older than 4.14.
- STS support for non-AWS cloud providers (Azure Workload Identity, GCP WIF tracked in PROJQUAY-7729).
- Automatic IAM role or IAM policy creation in the customer's AWS account.
- STS for unmanaged ObjectStorage in this iteration (see analysis below).
- Changes to the Quay application (`quay/quay`).

## Proposal

### How CCO + STS Works (Credential Flow)

Understanding the credential flow is critical to the design. This is distinct from the traditional CCO "Mint" mode, which produces long-lived IAM user keys.

**In STS/OIDC mode**, CCO does not create or rotate actual AWS credentials. Instead, it acts as a configuration broker:

1. The Quay operator creates a `CredentialRequest` CR that references the IAM role ARN and names the Quay app service account.
2. CCO reads the `CredentialRequest` and creates a Kubernetes `Secret` whose `credentials` key contains an AWS credentials file in the following format:

   ```ini
   [default]
   sts_regional_endpoints = regional
   role_arn = arn:aws:iam::123456789012:role/quay-s3-role
   web_identity_token_file = /var/run/secrets/openshift/serviceaccount/token
   ```

   This is **not** a static key — it is a pointer to a web identity token file and a role ARN.

3. The operator mounts this Secret into every Quay application pod as a volume at `/var/run/secrets/cloud/` and sets the env var `AWS_SHARED_CREDENTIALS_FILE=/var/run/secrets/cloud/credentials`.

4. OCP automatically projects the Quay app service account's OIDC-signed token at `/var/run/secrets/openshift/serviceaccount/token` (refreshed periodically by the kubelet).

5. When Quay's boto3 storage driver makes an S3 API call, boto reads the credentials file, sees it's a web identity configuration, reads the token from the token file, and calls `sts:AssumeRoleWithWebIdentity`. AWS validates the token against the cluster's OIDC endpoint and returns short-lived `AccessKeyId`/`SecretAccessKey`/`SessionToken`. **boto handles this transparently and re-fetches credentials when they near expiry** — no operator involvement is needed for rotation.

6. Quay's `config.yaml` storage configuration contains no credentials fields at all — boto uses the credential chain exclusively.

This is fundamentally different from the old proposal draft, where the operator would read static temporary credentials out of the CCO Secret and put them in `config.yaml`. That approach would require the operator to re-reconcile on every credential rotation. The file-mount approach means **Quay pods never need to restart when credentials rotate**.

### User Stories

#### Story 1 — ROSA Administrator installs Quay without static credentials

As a ROSA cluster administrator whose security policy prohibits static IAM user keys, I want to install Quay via OperatorHub, provide my pre-created IAM role ARN as an annotation on the `QuayRegistry` resource, and have the operator automatically configure Quay to use STS credentials — without any static keys appearing anywhere in the cluster.

#### Story 2 — Existing Quay installation on OCP retains static credential behavior

As an OCP cluster administrator running Quay with existing static S3 credentials from an ObjectBucketClaim, I want to upgrade the Quay operator without any behavior change; my static credentials continue to work unless I explicitly opt in to STS.

#### Story 3 — Operator surfaces a clear degraded state when STS configuration is incomplete

As a ROSA cluster administrator, if I annotate the `QuayRegistry` with a role ARN but CCO cannot provision the `CredentialRequest` (wrong ARN, OIDC provider not set up, OCP < 4.14), I want the `QuayRegistry` to report a `Degraded` condition with a human-readable message telling me exactly what to fix.

### Implementation Details

#### 1. IAM Role ARN Input

The administrator annotates the `QuayRegistry` with the role ARN:

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

If the annotation is absent, the operator follows the existing static-credential path (no change in behavior). The annotation approach matches the interim convention used by cert-manager-operator and other CCO-integrated operators while OLM evolves a first-class spec API for this.

#### 2. STS-Enabled Cluster Detection

Before creating a `CredentialRequest`, the operator must confirm that CCO is operating in STS/OIDC mode. The check sequence on each reconcile:

1. **Platform check**: Read `config.openshift.io/v1 Infrastructure cluster` and confirm `status.platformStatus.type == "AWS"`. Skip STS path entirely on non-AWS clusters.
2. **CCO mode check**: Read `operator.openshift.io/v1 CloudCredential cluster` and inspect `spec.credentialsMode`. If `credentialsMode` is `Mint` or `Passthrough`, CCO will attempt to create static IAM user keys, not web identity config — log a warning and fall back to static credentials.
3. **OIDC endpoint check**: Confirm `status.platformStatus.aws.resourceTags` or the Infrastructure CR carries an OIDC issuer URL, which is present on ROSA and OCP STS-enabled clusters.
4. **CRD availability check**: Confirm the `CredentialRequest` CRD exists (API discovery). On OCP < 4.14 or non-OCP environments it may be absent.

Only when all four checks pass does the operator proceed with the STS path.

New RBAC required (in addition to existing):

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

#### 3. CredentialRequest Lifecycle

The operator **creates the `CredentialRequest` at runtime** during reconciliation (not shipped statically in the bundle). This is the correct pattern for OLM operators that need the role ARN as user input. The `CredentialRequest` is created in the Quay registry's namespace:

```yaml
apiVersion: cloudcredential.openshift.io/v1
kind: CredentialRequest
metadata:
  name: quay-registry-aws
  namespace: quay-enterprise   # same namespace as QuayRegistry
  ownerReferences:
    - apiVersion: quay.redhat.com/v1
      kind: QuayRegistry
      name: example-registry
spec:
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: AWSProviderSpec
    stsIAMRoleARN: "arn:aws:iam::123456789012:role/quay-s3-role"
    statementEntries:
      - effect: Allow
        action:
          - s3:GetObject
          - s3:PutObject
          - s3:DeleteObject
          - s3:HeadObject
          - s3:AbortMultipartUpload
          - s3:ListBucketMultipartUploads
        resource: "arn:aws:s3:::quay-bucket/*"
      - effect: Allow
        action:
          - s3:ListBucket
          - s3:HeadBucket
          - s3:GetBucketLocation
          - s3:GetBucketCors
          - s3:PutBucketCors
        resource: "arn:aws:s3:::quay-bucket"
  secretRef:
    name: quay-aws-sts-credentials
    namespace: quay-enterprise
  serviceAccountNames:
    - quay-app    # the service account used by Quay application pods
```

The `stsIAMRoleARN` field (available since OCP 4.14's CCO) tells CCO to create a web-identity credentials file for this role rather than attempting to mint IAM user keys. The `statementEntries` serve as documentation of the required permissions (CCO does not create or modify IAM policies in STS mode — the administrator must have already attached equivalent permissions to the role).

The bucket name is available from the ObjectBucketClaim after it is bound, which occurs before the `CredentialRequest` is created — no ordering conflict.

The `CredentialRequest` is owned by the `QuayRegistry` resource so it is garbage-collected when the `QuayRegistry` is deleted.

#### 4. Credential File Mounting into Quay Pods

After CCO provisions the Secret (`quay-aws-sts-credentials`), the operator:

1. Adds a `volume` to the Quay app Deployment referencing the CCO Secret.
2. Adds a `volumeMount` in the Quay container at `/var/run/secrets/cloud/`.
3. Adds env var `AWS_SHARED_CREDENTIALS_FILE=/var/run/secrets/cloud/credentials` to the Quay container.
4. Ensures the Quay app pods' `ServiceAccount` (`quay-app`) is the one listed in `serviceAccountNames` of the `CredentialRequest`, so the projected OIDC token is issued for it.

The kubelet automatically projects a fresh OIDC token for the `quay-app` service account at `/var/run/secrets/openshift/serviceaccount/token`. No additional volume mount is needed — OCP handles this for any pod whose service account is on an OIDC-enabled cluster.

When boto needs to make an S3 call, it reads the credentials file, finds `web_identity_token_file`, reads the OIDC token from that path, and calls `sts:AssumeRoleWithWebIdentity`. The temporary credentials returned are cached in memory and refreshed by boto before they expire. Kubernetes rotates the OIDC token regularly; boto re-reads it on each credential refresh cycle.

#### 5. Quay config.yaml Changes

In `pkg/kustomize/secrets.go`, when `ctx.StorageSTSEnabled` is true, the generated `DISTRIBUTED_STORAGE_CONFIG` for `S3Storage` omits `aws_access_key_id` and `aws_secret_access_key` entirely:

```python
DISTRIBUTED_STORAGE_CONFIG:
  default:
    - S3Storage
    - s3_bucket: quay-bucket
      s3_region: us-east-1
      host: s3.amazonaws.com
      port: 443
      is_secure: true
      storage_path: /datastorage/registry
      # No access_key or secret_key — boto uses AWS_SHARED_CREDENTIALS_FILE
```

Quay's existing `S3Storage` backend in `storage/cloud.py` passes `aws_access_key_id` and `aws_secret_access_key` to the boto3 session only when they are non-empty. When omitted, boto falls through to the standard credential provider chain, which reads `AWS_SHARED_CREDENTIALS_FILE`. No changes to `quay/quay` are required.

#### 6. Degraded Condition

The operator watches for the `CredentialRequest` to reach `status.provisioned == true`. If this has not occurred within a configurable timeout (default: 5 minutes) after the annotation was added, the operator sets:

```
type:    Degraded
status:  True
reason:  CredentialRequestNotProvisioned
message: "CCO has not provisioned CredentialRequest quay-enterprise/quay-registry-aws.
          Verify that the IAM role ARN is correct, the cluster OIDC provider is
          configured, and CCO is running in STS mode (credentialsMode != Mint/Passthrough).
          See: https://docs.openshift.com/..."
```

The operator does not proceed to configuring Quay storage until the `CredentialRequest` is provisioned.

### Unmanaged ObjectStorage and STS

When `ObjectStorage: managed: false`, the user provides storage configuration directly in the `configBundleSecret`'s `config.yaml`. The operator does not create an ObjectBucketClaim, does not know the bucket name or endpoint, and does not generate storage configuration. This creates a fundamental difference for STS.

**Why the operator cannot create a CredentialRequest for unmanaged storage:**

- The `CredentialRequest`'s `statementEntries` should scope the `s3:*` permissions to the specific bucket ARN (`arn:aws:s3:::bucket-name/*`). The operator has no way to discover the bucket name from the user's config without parsing their opaque `config.yaml`.
- Using `resource: "*"` is possible but violates least-privilege and is unlikely to be acceptable to the security review process.
- The user managing their own storage config implies they also manage their own credentials — operator intervention in this flow is architecturally inconsistent.

**Options for users with unmanaged ObjectStorage who want STS:**

| Approach | How | When to use |
|---|---|---|
| **EC2 instance profile / IRSA annotation** | Annotate the Quay app `ServiceAccount` with `eks.amazonaws.com/role-arn: <arn>`. OCP injects the OIDC token automatically. Provide `S3Storage` config in `config.yaml` with no credentials. boto resolves via IRSA. | ROSA/OSD. User fully controls the IAM role and trust policy. |
| **`STSS3Storage` with cross-account role** | Use Quay's built-in `STSS3Storage` storage class in `config.yaml`, providing `sts_role_arn`, `sts_user_access_key`, and `sts_user_secret_key`. This uses `sts:AssumeRole` (not web identity) to obtain temporary credentials, which boto refreshes automatically. | When an IAM user with assume-role permission is acceptable. Not suitable for ROSA environments that prohibit all static IAM keys. |
| **`AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE` via Override** | Set env vars on the Quay app deployment via the `QuayRegistry.spec.components[ObjectStorage].overrides.env` field (if implemented). Use `S3Storage` with no credentials in `config.yaml`. | Advanced users who want IRSA without the `eks.amazonaws.com` annotation. |

**Recommendation**: In the initial implementation, document option 1 (IRSA service account annotation) for unmanaged storage on ROSA. The operator does not need code changes to support this path — the user annotates the service account manually and provides a credentials-free `config.yaml`. Future iterations can add an operator-assisted path after the managed storage flow is validated.

### Required IAM Permissions

Derived from static analysis of `storage/cloud.py` in `quay/quay`:

| IAM Action | S3 Operation | Purpose |
|---|---|---|
| `s3:GetObject` | `get_object()` | Download blobs and manifests |
| `s3:PutObject` | `put_object()` | Upload blobs and manifests |
| `s3:DeleteObject` | `delete_object()` | Delete blobs during garbage collection |
| `s3:HeadObject` | `head_object()` | Check object existence and size |
| `s3:ListBucket` | `list_objects_v2()` | Enumerate objects for cleanup |
| `s3:HeadBucket` | `head_bucket()` | Verify bucket accessibility at startup |
| `s3:GetBucketLocation` | implicit in presigned URL generation | Determine bucket region |
| `s3:AbortMultipartUpload` | `abort_multipart_upload()` | Clean up failed layer uploads |
| `s3:ListBucketMultipartUploads` | `list_multipart_uploads()` (via paginator) | Find and clean up stale multipart uploads |
| `s3:GetBucketCors` | `get_bucket_cors()` | Read CORS configuration |
| `s3:PutBucketCors` | `put_bucket_cors()` | Set CORS configuration for browser-based pushes |

The multipart upload actions (`s3:CreateMultipartUpload`, `s3:UploadPart`, `s3:CompleteMultipartUpload`) are also needed; boto calls them via the `initiate_multipart_upload` / `upload_part` / `complete` APIs. These are covered by the object-level `s3:PutObject`-family actions in most AWS managed policies but should be listed explicitly for clarity.

`s3:PutBucketCors` is only required during Quay startup when CORS configuration is being set. It is included in the role policy for simplicity; operators with strict policies may choose to separate it into a one-time setup role.

**Example IAM policy document:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:HeadObject",
        "s3:AbortMultipartUpload",
        "s3:ListBucketMultipartUploads",
        "s3:CreateMultipartUpload",
        "s3:UploadPart",
        "s3:CompleteMultipartUpload"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:HeadBucket",
        "s3:GetBucketLocation",
        "s3:GetBucketCors",
        "s3:PutBucketCors"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    }
  ]
}
```

The IAM role's trust policy must reference the cluster's OIDC provider and restrict to the `quay-app` service account:

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

### RHEL-Based Quay Deployments

For Quay running on bare-metal or VMs outside OCP (no CCO, no projected OIDC tokens):

**Option A — EC2 Instance Profile (recommended for AWS-hosted VMs):** Attach an IAM instance profile with the permissions listed above to the EC2 instance running Quay. boto's credential chain automatically uses the instance metadata service (IMDSv2). No credentials in `config.yaml`.

**Option B — `STSS3Storage` (cross-account assume-role):** Quay ships a purpose-built `STSS3Storage` class in `storage/cloud.py`. Configure it in `config.yaml`:

```yaml
DISTRIBUTED_STORAGE_CONFIG:
  default:
    - STSS3Storage
    - sts_role_arn: arn:aws:iam::123456789012:role/quay-s3-role
      sts_user_access_key: AKIAIOSFODNN7EXAMPLE
      sts_user_secret_key: wJalrXUtnFEMI/...
      s3_bucket: quay-bucket
      s3_region: us-east-1
      storage_path: /datastorage/registry
```

`STSS3Storage` calls `sts:AssumeRole` using the provided IAM user credentials and automatically refreshes the temporary credentials before expiry. This still uses static IAM user keys (just scoped to `sts:AssumeRole`), so it is not acceptable on ROSA but is a viable improvement over putting long-lived S3 keys directly in config.

**Option C — `AWS_ROLE_ARN` + Web Identity Token File:** If the host or container can obtain an OIDC token (e.g., from an external OIDC provider), set `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` and use standard `S3Storage` with no credentials in `config.yaml`. boto resolves credentials via `AssumeRoleWithWebIdentity` automatically.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| CCO absent (OCP < 4.14, non-OCP) | Operator detects missing `CredentialRequest` CRD or wrong CCO mode at startup and skips STS path entirely; no crash |
| CCO in Mint/Passthrough mode | Detection step (see above) falls back to static credentials and logs a warning |
| Role ARN wrong or trust policy misconfigured | `Degraded` condition with actionable message; operator retries on next reconcile |
| OIDC token not projected (service account not OIDC-enabled) | boto fails with clear AuthorizationError on first S3 call; operator surfaces this in status |
| Regression on non-STS upgrades | STS path requires the annotation; upgrades without the annotation are fully unaffected |
| Multipart upload in flight when credentials rotate | boto refreshes credentials mid-upload transparently; the same session token is used for the duration of the `UploadPart` calls within a single upload and is valid for the session duration (1h by default, configurable) |

## Design Details

### Graduation Criteria

#### Dev Preview

- Operator creates and manages the `CredentialRequest` on annotated `QuayRegistry` resources where managed ObjectStorage is used.
- CCO-provisioned credentials file is mounted into Quay pods.
- Image push and pull succeed on a ROSA cluster with no static AWS credentials in any Secret.
- Graceful fallback to static credentials when annotation is absent.
- `Degraded` condition set when `CredentialRequest` not provisioned.

#### Tech Preview

- E2E kuttl tests pass on OCP 4.14+ STS-enabled clusters in CI.
- IAM policy document reviewed by security team and published in operator documentation.
- CSV annotation `token-auth-aws: "true"` verified with OperatorHub metadata validation tooling.
- Unmanaged ObjectStorage IRSA workaround documented.

#### GA

- Upgrade path from static credentials to STS documented and tested (add annotation to existing `QuayRegistry`, operator migrates without downtime).
- RHEL-based deployment guidance (`STSS3Storage`, instance profile) published in official Quay documentation.
- Metric or alert for `CredentialRequestNotProvisioned` available in the Quay operator's metrics endpoint.

### Test Plan

- **Unit**: Verify `CredentialRequest` is created with correct `stsIAMRoleARN` and `serviceAccountNames` when annotation present; not created when absent.
- **Unit**: Verify generated `config.yaml` omits `aws_access_key_id` / `aws_secret_access_key` when STS is enabled.
- **Unit**: Verify `Degraded` condition is set when `CredentialRequest.status.provisioned` is false past the timeout.
- **Unit**: Verify cluster detection logic (platform, CCO mode, OIDC issuer) correctly gates the STS path.
- **Integration**: With a CCO mock, verify full reconcile loop produces Quay pods with the volume mount and env var set; verify the CCO Secret is watched correctly.
- **E2E (kuttl)**: On a live ROSA cluster, push and pull images; verify no AWS credentials appear in any Secret or `config.yaml`.
- **Regression**: On a standard OCP cluster (Mint mode) without the annotation, verify operator behavior is identical to pre-enhancement.

### Upgrade / Downgrade Strategy

- **Upgrade** (adding STS to an existing install): Add the annotation to the `QuayRegistry`. The operator creates the `CredentialRequest` on the next reconcile, waits for CCO to provision it, then mounts the credentials file and updates the storage config. Quay pods are rolled out with the new config. Static credentials from the OBC Secret are no longer used.
- **Downgrade** (removing STS): Remove the annotation. The operator deletes the `CredentialRequest` (via owner reference GC) and reverts to static credential sourcing from the OBC Secret on the next reconcile. Quay pods are rolled out to remove the volume mount and restore `AccessKey`/`SecretKey` in config.

### Version Skew Strategy

The `CredentialRequest` CRD is provided by CCO, which ships as part of OCP. The operator discovers CRD availability at startup via API discovery and skips the STS path when the CRD is absent. This prevents crashes on older or non-OCP clusters.

## Implementation History

- 2023-07-19 PROJQUAY-5850 filed; feasibility investigation completed.

## Drawbacks

- Adds a CCO and OIDC dependency for the STS path. On clusters where CCO is misbehaving or the OIDC provider is misconfigured, Quay storage is unavailable until the administrator resolves it.
- The annotation input mechanism is unconventional; a `QuayRegistry.spec.cloudCredentialsRef` or similar field would be more idiomatic. This can be added in a follow-up without breaking the annotation-based path.
- `s3:PutBucketCors` in the ongoing role policy is broader than strictly necessary for day-to-day operations, but separating it into a setup-only policy increases operational complexity.

## Alternatives

- **`eks.amazonaws.com/role-arn` ServiceAccount annotation only (no CCO)**: Simpler, no CCO dependency, but requires the administrator to annotate a service account that the operator manages, which can be overwritten on reconcile. Does not integrate with OCP console STS workflow.
- **`STSS3Storage` (cross-account assume-role)**: Already in the Quay application; usable today but still requires static IAM user credentials for the initial `AssumeRole` call, so it doesn't satisfy ROSA security requirements.
- **CCO "Manual" mode**: Administrator pre-creates the `CredentialRequest` before installing the operator; the operator reads the resulting Secret. Adds an out-of-band installation step that increases complexity for users.

## Infrastructure Needed

- A ROSA or OCP 4.14+ STS-enabled cluster for E2E testing in CI.
- IAM role with the permissions documented above, with the CI cluster's OIDC provider configured in the trust policy.
- `ccoctl` or equivalent tooling in CI to manage the OIDC provider setup during test cluster provisioning.
