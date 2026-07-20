---
title: TLS for Operator-Managed PostgreSQL
authors:
  - "@bcaton"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-06-02
last-updated: 2026-06-02
status: implementable
see-also:
  - "/enhancements/tls-managed-component.md"
---

# tls-managed-postgres

Enable TLS encryption for operator-managed PostgreSQL deployments used by
Quay Registry and Clair, satisfying regulatory requirements for encryption
in transit within the cluster network.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA

## Open Questions

> ~~1. Should certificate rotation be automated for self-signed certs, or is
>    manual rotation (delete managed keys entry, trigger reconcile) acceptable
>    for the initial implementation?~~
>
> **Resolved**: Manual rotation for the initial implementation. Users delete
> the TLS entries from the managed keys Secret and trigger a reconcile. The
> operator detects the CA mismatch, regenerates certificates, and performs a
> coordinated restart of all affected components (see "Certificate Mismatch
> Detection and Recovery"). Users requiring automated rotation should use
> cert-manager with `secretRef`.
>
> ~~2. Should the operator emit Kubernetes Events for TLS-related state
>    changes (cert generated, cert rotation, TLS errors)?~~
>
> **Resolved**: Yes. The operator emits the following Kubernetes Events on
> the `QuayRegistry` object:
>
> | Event | Type | When |
> |-------|------|------|
> | `TLSCertificateGenerated` | Normal | Self-signed CA and server certificate created for the first time |
> | `TLSCertificateRegenerated` | Warning | CA certificate was regenerated due to missing/modified managed keys Secret |
> | `TLSConfigurationApplied` | Normal | TLS configuration successfully applied to PostgreSQL and client deployments |
> | `TLSValidationFailed` | Warning | User-provided certificate validation failed (missing Secret, bad cert, expired, etc.) |

## Summary

The Quay Operator deploys managed PostgreSQL instances for both Quay Registry
and Clair as a convenience option. Currently, all database connections are
unencrypted. Customers in regulated industries (financial services, healthcare,
government) require encryption in transit for **all** connections, including
those inside the OpenShift SDN. This is a regulatory and audit requirement that
blocks deployment of Quay to their platforms.

This enhancement adds opt-in TLS support for operator-managed PostgreSQL via
the existing component override mechanism in the `QuayRegistry` CRD:

- PostgreSQL pods are configured to require TLS connections.
- Quay and Clair `DB_URI`/connection strings are automatically updated with
  TLS parameters (`sslmode=verify-full`).
- Certificate provisioning supports both operator-generated self-signed
  certificates and user-provided certificates (including those managed by
  cert-manager).

This is distinct from the existing `tls` managed component
(`tls-managed-component.md`), which handles Quay's external HTTPS endpoint.
This enhancement addresses internal database transport encryption.

## Motivation

Customers in regulated industries such as financial services, healthcare,
and government require encryption for all traffic, including internal
communication within the OpenShift SDN. This is a regulatory and audit
requirement that blocks deployment of workloads to their platforms and
hinders adoption (RFE-7835, RFE-9068).

Quay already supports TLS connections to **external** PostgreSQL via `DB_URI`
configuration (e.g., `?sslmode=verify-full&sslrootcert=/path/to/ca.crt`).
However, when using the operator-managed PostgreSQL convenience deployment,
there is no way to enable TLS. Customers who require encryption in transit are
forced to deploy and manage their own external PostgreSQL, significantly
increasing operational burden.

### Goals

- **Encryption in transit**: All connections between Quay/Clair and their
  managed PostgreSQL instances are TLS-encrypted when the feature is enabled.
- **Opt-in via CRD**: TLS is configured through the existing `overrides` field
  on the `postgres` and `clairpostgres` components, maintaining backward
  compatibility.
- **Self-signed certificates**: The operator generates self-signed CA and
  server certificates for simple deployments with no external dependencies.
- **User-provided certificates**: Users can reference a Kubernetes Secret
  containing custom certificates, enabling integration with enterprise PKI or
  cert-manager.
- **Backward compatibility**: Existing deployments are completely unaffected.
  TLS is only enabled when explicitly requested.
- **Migration support**: Existing deployments can enable TLS without data loss.
  The operator detects the configuration change and updates connection strings
  and PostgreSQL configuration accordingly.

### Non-Goals

- **Mutual TLS (mTLS)**: Client certificate authentication for database
  connections. The initial implementation uses server-side TLS with hostname
  verification (`sslmode=verify-full`). Client certificate authentication,
  where the database verifies the client's identity via certificate, is out
  of scope.
- **Automatic certificate rotation**: Self-signed certificates are generated
  with long validity (10 years). Automated rotation is out of scope for the
  initial implementation. Users who need rotation should use cert-manager with
  `secretRef`.
- **Direct cert-manager integration**: The operator does not create
  `Certificate` CRs. Users can configure cert-manager independently to
  populate a Secret, then reference it via `secretRef`.
- **TLS for Redis**: Redis connection encryption is a separate concern and
  not addressed here.
- **TLS for external PostgreSQL**: Users connecting to external databases
  already configure TLS via `DB_URI` parameters. This enhancement only
  addresses operator-managed instances.

## Proposal

### User Stories

#### Story 1: Regulated Environment Deployment

As a platform engineer in a regulated industry, I need all internal traffic
encrypted so that I can pass security audits. I want to deploy Quay with
managed PostgreSQL and enable TLS with minimal configuration:

```yaml
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: my-registry
spec:
  components:
  - kind: postgres
    managed: true
    overrides:
      tls:
        enabled: true
  - kind: clairpostgres
    managed: true
    overrides:
      tls:
        enabled: true
```

The operator generates self-signed certificates, configures PostgreSQL to
require TLS, and updates all connection strings automatically.

#### Story 2: Enterprise PKI Integration

As a security engineer with an existing PKI infrastructure, I want to use
my organization's certificates for database TLS. I create a Secret with my
certificates and reference it:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-postgres-tls
data:
  ca.crt: <base64-encoded CA certificate>
  tls.crt: <base64-encoded server certificate>
  tls.key: <base64-encoded server private key>
---
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: my-registry
spec:
  components:
  - kind: postgres
    managed: true
    overrides:
      tls:
        enabled: true
        secretRef:
          name: my-postgres-tls
```

#### Story 3: cert-manager Integration

As a cluster administrator using cert-manager, I want automated certificate
lifecycle management for my database connections. I create a cert-manager
`Certificate` CR that populates a Secret, then reference that Secret:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: postgres-tls
spec:
  secretName: postgres-tls-cert
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
  dnsNames:
  - my-registry-quay-database
  - my-registry-quay-database.my-namespace.svc.cluster.local
---
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: my-registry
spec:
  components:
  - kind: postgres
    managed: true
    overrides:
      tls:
        enabled: true
        secretRef:
          name: postgres-tls-cert
```

#### Story 4: Enabling TLS on Existing Deployment

As an operator of an existing Quay deployment, I want to enable TLS for my
managed PostgreSQL without redeploying. I update my `QuayRegistry` CR to add
the TLS override. The operator detects the change, generates certificates,
reconfigures PostgreSQL, updates connection strings, and restarts the
affected pods. The brief restart is expected (documented) and there is no
data loss.

**Acceptance criteria**:
- Enabling TLS causes a single restart cycle (PostgreSQL pod recreated,
  then Quay/Clair pods restart due to config change). Expected downtime is
  under 2 minutes, consistent with any managed PostgreSQL config change.
- After restart, `kubectl exec <pg-pod> -- psql -c "SHOW ssl"` returns
  `on`, and the `QuayRegistry` status shows all component conditions True.
- If the TLS-enabled PostgreSQL pod fails to start (e.g., cert file
  permission error), the operator sets a `ComponentDegraded` status
  condition. The user can roll back by removing `overrides.tls` or setting
  `enabled: false`.

#### Story 5: Diagnosing TLS Misconfiguration

As a platform engineer, when I reference an invalid or misconfigured TLS
certificate Secret in my `QuayRegistry` CR, I need a clear error on the CR
status so I can diagnose and fix the problem without reading operator pod
logs.

```yaml
spec:
  components:
  - kind: postgres
    managed: true
    overrides:
      tls:
        enabled: true
        secretRef:
          name: wrong-secret-name  # typo or missing Secret
```

The operator validates the referenced Secret during reconciliation and
surfaces specific errors:

- `kubectl get quayregistry my-registry -o jsonpath='{.status.conditions}'`
  shows `ComponentDegraded: postgres — TLS secret "wrong-secret-name" not
  found`.
- `kubectl get events` shows a `TLSValidationFailed` warning Event with
  details about the specific issue (missing Secret, missing key, cert/key
  mismatch, or expired certificate).
- The existing deployment is not modified — if TLS was previously working,
  it continues to work. If this is a new deployment, PostgreSQL starts
  without TLS until valid certificates are provided.

### Implementation Details/Notes/Constraints

#### CRD Changes

A new `TLSOverride` struct is added to the existing `Override` type:

```go
type TLSOverride struct {
    Enabled   bool                             `json:"enabled"`
    SecretRef *corev1.LocalObjectReference     `json:"secretRef,omitempty"`
}
```

The `Override` struct gains a `TLS *TLSOverride` field. A new
`supportsTLSOverride` list restricts TLS overrides to `postgres` and
`clairpostgres` components. This follows the established pattern used by
all other overrides (volumeSize, storageClassName, env, etc.).

The `TLS *TLSOverride` pointer on `Override` (nil vs non-nil) provides the
"not configured" vs "explicitly configured" distinction, so `Enabled bool`
(not `*bool`) is intentional — `tls: {}` and `tls: {enabled: false}` are
both treated as "TLS disabled."

**CRD validation rules** (CEL):

| Rule | CEL Expression | Error Message |
|------|----------------|---------------|
| TLS override only on supported components | `!(self.kind in ['postgres', 'clairpostgres']) ? !has(self.overrides.tls) : true` | `TLS override is only supported on postgres and clairpostgres components` |
| `secretRef` requires `enabled: true` | `!has(self.overrides.tls.secretRef) \|\| self.overrides.tls.enabled` | `tls.secretRef requires tls.enabled to be true` |

#### Certificate Generation

When TLS is enabled without a `secretRef`, the operator generates a
self-signed CA and server certificate using ECDSA P-256 with 10-year
validity. The server certificate includes SANs for all in-cluster DNS
names:

- `<name>-quay-database`
- `<name>-quay-database.<namespace>`
- `<name>-quay-database.<namespace>.svc`
- `<name>-quay-database.<namespace>.svc.cluster.local`
- `localhost`

Generated certificates are persisted in the managed keys Secret to survive
reconcile loops, using the following key names:

| Managed Keys Entry | Component | Description |
|--------------------|-----------|-------------|
| `postgres-tls-ca.crt` | postgres | CA certificate (PEM) |
| `postgres-tls-server.crt` | postgres | Server certificate signed by the CA (PEM) |
| `postgres-tls-server.key` | postgres | Server private key (PEM, ECDSA P-256) |
| `clairpostgres-tls-ca.crt` | clairpostgres | CA certificate (PEM) |
| `clairpostgres-tls-server.crt` | clairpostgres | Server certificate signed by the CA (PEM) |
| `clairpostgres-tls-server.key` | clairpostgres | Server private key (PEM, ECDSA P-256) |

Each component gets its own CA, so `postgres` and `clairpostgres` TLS can
be enabled, disabled, and rotated independently. These key names are
prefixed with the component kind to avoid collision with existing managed
keys entries (e.g., `postgres-password`, `clair-database-password`).

#### Certificate Mismatch Detection and Recovery

If the managed keys Secret is deleted or its TLS entries are modified, the
operator regenerates the CA and server certificates on the next reconcile.
This creates a mismatch: running PostgreSQL pods have the old server
certificate, while newly created Secrets contain the new CA certificate.
Any Quay/Clair pod that restarts would pick up the new CA and fail to
verify the old server certificate, causing a partial or full outage.

To prevent this split-brain state, the operator performs a CA mismatch
check during reconciliation:

1. **Detection**: After generating or loading certificates, the operator
   compares the CA certificate fingerprint against the CA currently stored
   in the `postgresql-ca` Secret (if it exists). A mismatch indicates the
   CA was regenerated and all running pods have stale certificates.

2. **Coordinated restart**: When a CA mismatch is detected, the operator
   annotates all affected Deployments with a `quay-operator/tls-cert-hash`
   annotation containing the new CA fingerprint. This forces Kubernetes to
   roll all pods — PostgreSQL first (it has no dependencies), then
   Quay/Clair (which may briefly crash-loop until PostgreSQL is ready with
   the new cert, consistent with standard `Recreate` strategy behavior).

3. **Status condition**: The operator sets a status condition on the
   `QuayRegistry` CR:
   `CertificateRotated: postgres — CA certificate was regenerated; all
   components restarting`.

4. **Event**: The operator emits a warning Event:
   `TLSCertificateRegenerated: managed keys Secret was missing or modified;
   certificates were regenerated and all TLS components are restarting`.

This mechanism also handles intentional manual rotation: a user can delete
the TLS entries from the managed keys Secret and trigger a reconcile to
force certificate regeneration. The operator detects the mismatch, rolls
all components, and the system converges on the new certificates with a
brief restart.

#### PostgreSQL Server Configuration

An init container (`postgres-tls-init`) is injected into the PostgreSQL
deployment when TLS is enabled. It:

1. Copies certificates from a projected volume into the PostgreSQL data
   directory (`$PGDATA/userdata/`) with correct permissions (key: 0600).
2. Patches `postgresql.conf` to add SSL directives if not already present.

This approach works for both new deployments (fresh PVC) and existing
deployments where `postgresql.conf` is already baked into the data
directory. The sclorg PostgreSQL image's `postgresql.conf.sample` is only
used during initial PVC setup, so a ConfigMap-only approach would not work
for existing deployments.

#### Connection String Updates

Connection strings are modified using proper URL parsing, not string
concatenation, to avoid producing invalid URIs when existing query
parameters are present.

**Quay**: The managed `DB_URI` is parsed using Go's `net/url.Parse()`.
TLS parameters are merged into the existing query values:

```go
u, err := url.Parse(existingDBURI)
// if parsing fails: set ComponentDegraded, do not modify the URI
q := u.Query()
q.Set("sslmode", "verify-full")
q.Set("sslrootcert", tlsCAMountPath) // e.g., "/tls/ca.crt"
u.RawQuery = q.Encode()
newDBURI := u.String()
```

This approach correctly handles URIs that already have query parameters
(e.g., `?connect_timeout=10`), preserves the password (including
URL-encoded special characters) automatically via the parsed `Userinfo`
component, and is idempotent — running it twice produces the same result.
On downgrade, the same parse/delete/encode pattern removes TLS parameters:

```go
q := u.Query()
q.Del("sslmode")
q.Del("sslrootcert")
u.RawQuery = q.Encode()
```

If `net/url.Parse()` fails on a malformed URI, the operator sets a
`ComponentDegraded` status condition and does not modify the existing
`DB_URI`, preserving the current working connection rather than writing a
broken one.

The CA certificate mount path (`/tls/ca.crt` for Quay,
`/clair-db-tls/ca.crt` for Clair) is defined as a constant in the
middleware and referenced consistently in both the connection string and
the projected volume mount, so the two cannot drift apart.

The existing `postgres-certs` projected volume (already present in the
Quay app, mirror, and upgrade job deployments with `optional: true` on the
Secret sources) automatically picks up the CA certificate.

**Clair**: The hardcoded `sslmode=disable` in the Clair config generation
is replaced with conditional logic that sets `sslmode=verify-full` and
`sslrootcert` when TLS is enabled. The Clair connection string uses
key-value format:

```
# Without TLS (current):
host=my-registry-clair-database port=5432 dbname=clair user=clair password=... sslmode=disable

# With TLS:
host=my-registry-clair-database port=5432 dbname=clair user=clair password=... sslmode=verify-full sslrootcert=/clair-db-tls/ca.crt
```

A new `clair-db-tls` projected volume is injected into the Clair
deployment to provide the CA certificate at the path referenced in the
connection string.

#### Client-Side Certificate Distribution

The operator creates a `postgresql-ca` Secret containing the CA certificate.
Every Deployment and Job that connects to managed PostgreSQL must have the
CA certificate mounted. The following table enumerates all database
consumers and their CA cert volume configuration:

| Consumer | Connects to | Volume | Mount Path | Existing? |
|----------|-------------|--------|------------|-----------|
| Quay app | `postgres` | `postgres-certs` (projected, `optional: true`) | `/run/secrets/postgresql/ca.crt` | Yes — verify in `quay-app.deployment.yaml` |
| Quay mirror | `postgres` | `postgres-certs` (projected, `optional: true`) | `/run/secrets/postgresql/ca.crt` | Yes — verify in `quay-mirror.deployment.yaml` |
| Quay upgrade job | `postgres` | `postgres-certs` (projected, `optional: true`) | `/run/secrets/postgresql/ca.crt` | Yes — verify in `quay-upgrade.job.yaml` |
| Clair | `clairpostgres` | `clair-db-tls` (projected) | `/clair-db-tls/ca.crt` | No — injected via middleware when clairpostgres TLS is enabled |

> **Implementation note**: The "Existing? Yes" entries above must be
> verified against the current operator source code during implementation.
> If any projected volume does not already exist, the middleware must be
> updated to inject it (following the same pattern used for Clair).

For consumers with existing projected volumes, the `optional: true` flag
on the Secret source means the volume mounts successfully even when the
`postgresql-ca` Secret does not exist (i.e., when TLS is disabled). When
TLS is enabled, the operator creates the Secret and the projected volume
automatically picks up the CA cert on the next pod restart.

For Clair, which has no existing postgres cert volumes, a new projected
volume (`clair-db-tls`) is injected via middleware when Clair database TLS
is enabled.

#### Error Handling for User-Provided Certificates

When `secretRef` is specified, the operator validates the referenced Secret
during reconciliation **before** applying any configuration changes to
deployments. If validation fails, the operator sets a degraded status
condition on the `QuayRegistry` CR, emits a Kubernetes Event, and does not
modify the current deployment state (leaving the existing working
configuration in place).

| Condition | Operator Behavior | Status Condition |
|-----------|-------------------|------------------|
| Secret not found | Requeue reconcile; do not modify deployments | `ComponentDegraded: postgres — TLS secret "<name>" not found` |
| Secret missing required keys | Do not modify deployments | `ComponentDegraded: postgres — TLS secret "<name>" missing required key "<key>" (expected: ca.crt, tls.crt, tls.key)` |
| Certificate/key mismatch | Do not modify deployments | `ComponentDegraded: postgres — TLS certificate and private key in secret "<name>" do not match` |
| Certificate expired | Emit warning event; do not modify deployments | `ComponentDegraded: postgres — TLS certificate in secret "<name>" expired on <date>` |
| Secret deleted after deployment | Emit warning event; set degraded condition | `ComponentDegraded: postgres — TLS secret "<name>" has been deleted` |

Validation is performed using Go's `tls.X509KeyPair()` for cert/key
matching and `x509.Certificate.NotAfter` for expiry checking.

**Secret watching**: The operator watches Secrets referenced by `secretRef`
and triggers reconciliation on changes. This enables cert-manager rotation
workflows: when cert-manager updates the Secret with a rotated certificate,
the operator detects the change, re-validates the new certificate, and
updates the projected volumes to pick up the new cert. The PostgreSQL pod
restarts to load the updated certificate.

**Secret schema contract**: User-provided Secrets must contain the
following keys in PEM format:

| Key | Required | Description |
|-----|----------|-------------|
| `ca.crt` | Yes | CA certificate (or full chain) used to verify the server certificate. Mounted into Quay/Clair pods for `sslrootcert`. |
| `tls.crt` | Yes | Server certificate presented by PostgreSQL to connecting clients. May include intermediate certificates (leaf first). |
| `tls.key` | Yes | Private key corresponding to `tls.crt`. Must not be passphrase-protected. |

This key naming convention is consistent with Kubernetes TLS Secret type
(`kubernetes.io/tls`) and cert-manager output format, with the addition of
`ca.crt` which cert-manager also populates by default.

### Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| sclorg image doesn't re-read `postgresql.conf` from the ConfigMap after initial PVC setup | Init container patches the copy in the PVC data directory directly |
| Self-signed certificates expire | 10-year validity for initial implementation; users needing rotation use cert-manager via `secretRef` |
| Enabling TLS on existing deployment causes brief downtime | PostgreSQL deployment uses `Recreate` strategy (existing behavior for any config change); document the expected brief restart |
| Init container permission issues on OpenShift | Uses the same image as the main postgres container, which already runs successfully under the restricted SCC |
| PEM certificate data increases managed keys Secret size | ECDSA P-256 certs are ~1KB each; well under etcd's 1MB Secret limit |
| DB_URI transition breaks existing deployments | TLS params merged via `net/url.Parse()` (not string concatenation), preserving existing query parameters and URL-encoded passwords. If URI parsing fails, operator sets `ComponentDegraded` and does not modify the URI. |
| User provides invalid certificates via `secretRef` | Operator validates cert chain at reconcile time (key pair matching, expiry, required keys) and surfaces specific errors via status conditions and Kubernetes Events. Deployments are not modified until validation passes. |
| Referenced `secretRef` Secret is deleted or modified | Operator watches referenced Secrets and triggers reconciliation on changes. Deletion sets a degraded condition; updates trigger re-validation and cert rollout. |
| Managed keys Secret deleted or TLS entries modified | Operator detects CA mismatch during reconciliation, regenerates certificates, annotates all affected Deployments to force a coordinated restart, and emits a `TLSCertificateRegenerated` warning Event. Brief downtime during restart is expected and documented. |

## Design Details

### Test Plan

**Unit Tests**:
- CRD validation: TLS override accepted on postgres/clairpostgres, rejected
  on unsupported components (redis, clair, etc.)
- CRD validation: `tls: {}` (without `enabled: true`) does not enable TLS
- CRD validation: `tls.secretRef` without `tls.enabled: true` is rejected
- Self-signed certificate generation: valid cert chain, correct SANs, PEM
  encoding, CA signs server cert, ECDSA P-256 key type
- Middleware TLS injection: postgres deployment gets TLS `-c` args and
  projected volume when override is enabled; no changes when disabled
- Connection string (Quay): DB_URI includes `sslmode=verify-full` when TLS
  enabled; existing query parameters are preserved; password with special
  characters is preserved
- Connection string (Clair): connstring uses `sslmode=verify-full` and
  `sslrootcert` when TLS enabled; uses `sslmode=disable` when disabled
- DB_URI parsing: malformed URI sets `ComponentDegraded`, does not modify URI
- `secretRef` validation: missing Secret, missing keys, cert/key mismatch,
  and expired cert each produce the correct `ComponentDegraded` condition
- Certificate mismatch detection: CA fingerprint change triggers Deployment
  annotation update
- Mixed-mode: TLS enabled on `postgres` but not `clairpostgres` (and vice
  versa) produces correct per-component configuration

**E2E Tests** (Chainsaw):
- Deploy QuayRegistry with TLS enabled on both postgres and clairpostgres
- Assert postgres deployment has TLS `-c` args and `postgres-tls` volume
- Assert `postgresql-ca` and `postgres-tls` Secrets exist with expected keys
- Assert generated certificates have correct SANs and valid chain
- Assert all component status conditions are True
- Assert Quay app pods are running and healthy
- Assert Clair deployment has `clair-db-tls` volume
- Assert Clair pods are running and healthy
- Query `pg_stat_ssl` to confirm connections are encrypted end-to-end
- Deploy with `secretRef` pointing to a user-provided Secret; assert TLS
  works with custom certificates
- Deploy with `secretRef` pointing to a non-existent Secret; assert
  `ComponentDegraded` status condition is set
- Enable TLS on an existing non-TLS deployment (migration path); assert
  TLS is active after restart with no data loss
- Disable TLS on a TLS-enabled deployment (downgrade path); assert
  PostgreSQL starts without TLS and connections succeed
- Mixed-mode: enable TLS on `postgres` only; assert `clairpostgres` uses
  `sslmode=disable`

**Manual Verification**:
- `kubectl exec <pg-pod> -- psql -c "SHOW ssl"` returns `on`
- Quay app logs show successful database connection
- `psql` from within the Quay app pod using the generated DB_URI connects
  successfully over TLS
- `psql "postgresql://...?sslmode=disable"` from within the cluster
  confirms PostgreSQL accepts the connection (server-side enforcement of
  TLS-only is a follow-up; client-side `verify-full` ensures encryption)
- `openssl s_client -connect <pg-service>:5432 -starttls postgres`
  confirms certificate details (issuer, SANs, expiry)

### Graduation Criteria

#### Dev Preview

- TLS can be enabled via CRD override
- Self-signed certificates are generated and mounted
- User-provided certificate support via `secretRef` (including
  cert-manager-populated Secrets)
- Certificate validation and error reporting via status conditions and
  Events (see "Error Handling for User-Provided Certificates")
- PostgreSQL accepts TLS connections
- Quay and Clair connect over TLS with `sslmode=verify-full`
- Unit and e2e tests pass (including `secretRef` and negative test cases)

#### Tech Preview

- Migration path tested (enabling TLS on existing non-TLS deployment)
- Downgrade path tested (disabling TLS on a TLS-enabled deployment)
- Mixed-mode tested (`postgres` TLS enabled, `clairpostgres` TLS disabled,
  and vice versa)
- Certificate mismatch detection and coordinated restart verified
- Documentation published: deployment guide, migration procedure,
  certificate rotation runbook, troubleshooting guide

#### GA

- Upgrade testing complete: operator upgrade from N-1 to N with TLS
  enabled, and from N-1 (no TLS support) to N with TLS newly enabled
- Downgrade testing complete: operator downgrade from N to N-1 with TLS
  enabled (verify graceful degradation or documented manual steps)
- Certificate rotation verified end-to-end: manual rotation for self-signed
  certs, automated rotation via cert-manager with `secretRef`
- Performance baseline established: TLS connection overhead measured and
  documented (latency and throughput compared to non-TLS)
- At least 3 production or staging deployments validated during Tech
  Preview with no P1/P2 issues
- Feature is opt-in via `overrides.tls.enabled: true` (no feature gate
  required; the CRD field is available to all users)

### API Design

No new API endpoints. Changes are limited to the `QuayRegistry` CRD:

```yaml
spec:
  components:
  - kind: postgres        # or clairpostgres
    managed: true
    overrides:
      tls:
        enabled: true     # required, default false
        secretRef:        # optional
          name: string    # Secret containing ca.crt, tls.crt, tls.key
```

The `secretRef` here is on the `Override` (not the `Component`), because the
existing `Component.SecretRef` field has a CEL validation rule preventing its
use on managed components. The TLS override `secretRef` serves a different
purpose: it provides certificates for a managed component rather than
replacing the component entirely.

### Upgrade / Downgrade Strategy

**Upgrade (existing deployment, TLS disabled -> enabled)**:
1. User adds `overrides.tls.enabled: true` to their `QuayRegistry` CR.
2. Operator detects TLS is enabled but the persisted `DB_URI` lacks
   `sslmode` parameters.
3. Operator generates self-signed certificates (or reads user-provided ones).
4. Operator regenerates `DB_URI` with TLS parameters, preserving the
   existing password.
5. PostgreSQL deployment gets an init container and TLS volume, triggering
   a pod restart.
6. Quay/Clair pods restart due to config change.
7. No data loss; PVC contents are unchanged.

**Downgrade (TLS enabled -> disabled)**:
1. User removes `overrides.tls` or sets `enabled: false`.
2. Operator detects `DB_URI` has `sslmode` but TLS is disabled.
3. Operator regenerates `DB_URI` without TLS parameters.
4. Init container and TLS volume are removed from the postgres deployment.
5. PostgreSQL restarts without TLS.
6. Quay/Clair pods restart due to config change.

**No-op for existing deployments**:
The `TLS` field on `Override` is a pointer (`*TLSOverride`), defaulting to
`nil`. Existing `QuayRegistry` CRs without this field are completely
unaffected. The CRD change is additive and backward compatible.

### Version Skew Strategy

- The Quay application already supports TLS database connections via
  `DB_URI` parameters. No application-side changes are needed.
- Clair supports TLS via its `connstring` configuration. No Clair code
  changes are needed.
- The sclorg PostgreSQL images support SSL configuration via
  `postgresql.conf`. No image changes are needed.
- During an operator upgrade where the CRD adds the new field, existing CRs
  are unaffected (field defaults to nil/absent). The feature only activates
  when explicitly configured.

## Implementation History

- 2026-06-02: Initial proposal and implementation (PROJQUAY-11215)

## Drawbacks

- **Increased Secret size**: The managed keys Secret grows by ~3KB per
  enabled component (CA cert + server cert + key). This is negligible
  relative to the 1MB etcd limit.
- **Brief downtime on enable/disable**: Enabling or disabling TLS requires
  PostgreSQL and application pod restarts. This is consistent with any
  database configuration change and is documented.
- **Self-signed certificates require manual rotation**: The 10-year validity
  mitigates urgency, but environments requiring automated rotation must use
  cert-manager. A future enhancement could add operator-managed rotation.

## Alternatives

1. **Direct cert-manager integration**: The operator could create
   `Certificate` CRs and watch for the resulting Secrets. This was rejected
   because it introduces a hard dependency on cert-manager and increases
   operator complexity. The `secretRef` approach achieves the same outcome
   while keeping the operator cert-manager-agnostic.

2. **New top-level CRD field**: Instead of using the override mechanism, a
   dedicated `spec.databaseTLS` field could be added. This was rejected
   because it breaks the established pattern where all component
   configuration goes through `components[].overrides`, and would require
   special-case handling in the controller.

3. **TLS as a separate managed component**: Similar to the existing `tls`
   component (for HTTPS), a `databasetls` component could be added. This was
   rejected because database TLS is a property of the postgres component,
   not a standalone service, and adding a new component kind has broader CRD
   implications.

4. **Always-on TLS for managed PostgreSQL**: TLS could be enabled by default
   for all managed PostgreSQL deployments. This was rejected because it
   would be a breaking change for existing deployments and adds overhead for
   development/testing environments where encryption is not required.
