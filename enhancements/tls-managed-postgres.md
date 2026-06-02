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
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

> 1. Should certificate rotation be automated for self-signed certs, or is
>    manual rotation (delete managed keys entry, trigger reconcile) acceptable
>    for the initial implementation?
> 2. Should the operator emit Kubernetes Events for TLS-related state changes
>    (cert generated, cert rotation, TLS errors)?

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
  TLS parameters (`sslmode=verify-ca`).
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
  connections. The initial implementation uses server-side TLS with CA
  verification (`sslmode=verify-ca`), not `verify-full` with client certs.
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
reconcile loops.

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

**Quay**: The managed `DB_URI` is updated to include
`?sslmode=verify-ca&sslrootcert=/run/secrets/postgresql/ca.crt`. The
existing `postgres-certs` projected volume (already present in the Quay
app, mirror, and upgrade job deployments with `optional: true` on the
Secret sources) automatically picks up the CA certificate.

**Clair**: The hardcoded `sslmode=disable` in the Clair config generation
is replaced with conditional logic that sets `sslmode=verify-ca` and
`sslrootcert` when TLS is enabled. A new `clair-db-tls` projected volume
is injected into the Clair deployment to provide the CA certificate.

#### Client-Side Certificate Distribution

The operator creates a `postgresql-ca` Secret containing the CA certificate.
The Quay app, mirror, and upgrade job deployments already have projected
volumes that source from this Secret (with `optional: true`), so they
automatically pick up the CA cert without manifest changes to those
components.

For Clair, which has no existing postgres cert volumes, a new projected
volume (`clair-db-tls`) is injected via middleware when Clair database TLS
is enabled.

### Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| sclorg image doesn't re-read `postgresql.conf` from the ConfigMap after initial PVC setup | Init container patches the copy in the PVC data directory directly |
| Self-signed certificates expire | 10-year validity for initial implementation; users needing rotation use cert-manager via `secretRef` |
| Enabling TLS on existing deployment causes brief downtime | PostgreSQL deployment uses `Recreate` strategy (existing behavior for any config change); document the expected brief restart |
| Init container permission issues on OpenShift | Uses the same image as the main postgres container, which already runs successfully under the restricted SCC |
| PEM certificate data increases managed keys Secret size | ECDSA P-256 certs are ~1KB each; well under etcd's 1MB Secret limit |
| DB_URI transition breaks existing deployments | TLS params only appended when TLS override is explicitly enabled; password is preserved from existing URI during regeneration |

## Design Details

### Test Plan

**Unit Tests**:
- CRD validation: TLS override accepted on postgres/clairpostgres, rejected
  on unsupported components (redis, clair, etc.)
- Self-signed certificate generation: valid cert chain, correct SANs, PEM
  encoding
- Middleware TLS injection: postgres deployment gets init container and TLS
  volume when override is enabled; no changes when disabled
- Connection strings: Quay DB_URI includes sslmode when TLS enabled; Clair
  connstring uses conditional sslmode instead of hardcoded `disable`

**E2E Tests** (Chainsaw):
- Deploy QuayRegistry with TLS enabled on both postgres and clairpostgres
- Assert postgres deployment has `postgres-tls-init` init container
- Assert postgres deployment has `postgres-tls` volume
- Assert `postgresql-ca` and `postgres-tls` Secrets exist with expected keys
- Assert all component status conditions are True
- Assert Quay app pods are running and healthy

**Manual Verification**:
- `kubectl exec <pg-pod> -- psql -c "SHOW ssl"` returns `on`
- Quay app logs show successful database connection
- `psql` from within the Quay app pod using the generated DB_URI connects
  successfully over TLS

### Graduation Criteria

#### Dev Preview

- TLS can be enabled via CRD override
- Self-signed certificates are generated and mounted
- PostgreSQL accepts TLS connections
- Quay and Clair connect over TLS
- Unit and e2e tests pass

#### Tech Preview

- User-provided certificate support via `secretRef`
- Migration path tested (enabling TLS on existing deployment)
- Documentation published

#### GA

- Upgrade and downgrade testing complete
- Certificate rotation documented (manual for self-signed, automatic via
  cert-manager)
- Sufficient user feedback from Tech Preview
- Available by default (but still opt-in)

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
