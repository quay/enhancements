# PROJQUAY-6259: Simplified Read-Only Configuration

Status: Review-ready design\
Target: Quay 3.19\
Repos: `quay/quay`, `quay/quay-operator`

## Summary

This design adds a single operator-managed `spec.readOnly` toggle to the
`QuayRegistry` custom resource. The operator transitions Quay into and out of
read-only mode, manages the service key needed for boot and JWT signing, and
reports progress through status fields and conditions.

The feature is intended to make application-consistent backups practical:
administrators can block writes before taking database and object storage
backups, then return the registry to normal read-write mode.

### Lifecycle Overview

```text
  Admin sets                                           Admin sets
  spec.readOnly=true                                   spec.readOnly=false
       │                                                    │
       ▼                                                    │
  ┌─────────────┐   Secret created    ┌──────────────────┐  │
  │             │   Key imported      │                  │  │
  │  Normal ""  │──────────────────►  │  PreparingKey    │  │
  │             │                     │  (Rolling)       │  │
  └─────────────┘                     └────────┬─────────┘  │
       ▲                                       │            │
       │                          Keyserver    │            │
       │                          verified     ▼            │
       │                              ┌──────────────────┐  │
       │                              │                  │◄─┘ abort
       │                              │ EnteringReadOnly │
       │                              │ (Recreate)       │
       │                              └────────┬─────────┘
       │                                       │
       │                          All pods     │
       │                          rolled out   ▼
       │                              ┌──────────────────┐
       │                              │                  │  ReadOnly=True
       │                              │    ReadOnly      │  ReadOnlyActive
       │                              │    (Rolling)     │◄── BACKUP HERE
       │                              └────────┬─────────┘
       │                                       │
       │                          spec.readOnly│
       │                          = false      ▼
       │                              ┌──────────────────┐
       │                              │                  │
       │                              │ ExitingReadOnly  │
       │                              │ (Recreate)       │
       │                              └────────┬─────────┘
       │                                       │
       │                          Cleanup Job  │
       │                          expires key  │
       │                          Secret delete│
       │                                       │
       └───────────────────────────────────────┘

  Abort: spec.readOnly=false during PreparingKey or EnteringReadOnly
         → render normal config → rollout → cleanup Job → reset to ""

  Conditions:
    PreparingKey/EnteringReadOnly/ExitingReadOnly → ReadOnly=Unknown
    ReadOnly (stable)                             → ReadOnly=True/ReadOnlyActive
    Key degraded (expired/missing/unapproved)     → ReadOnly=True/ReadOnlyDegraded
    Cleanup failed                                → ReadOnly=False/ReadOnlyCleanupFailed
    Manual config detected / Override conflict    → ReadOnly=False (blocked)
```

## Goals

- Add `spec.readOnly` to `QuayRegistry`.
- Preserve backward compatibility with clusters that manually configure
  `REGISTRY_STATE: readonly`.
- Avoid manual SQL and config Secret edits for normal operator-managed
  usage. Migration from existing manual readonly may require config cleanup
  (see Manual Readonly Config).
- Persist readonly service key private material across pod restarts without
  storing private material in the Quay database.
- Keep the operator reconciling normally while read-only mode is active.
- Provide deterministic recovery from interrupted transitions.
- Clean up long-lived credentials when exiting or aborting read-only mode.

## Non-Goals

- Do not add dynamic config reload to Quay.
- Do not store private JWK material in the database.
- Do not add per-repository readonly behavior. Repository readonly is a
  separate Quay concept.
- Do not allow selected workers to keep writing during registry read-only mode.
- Do not let the operator directly mutate Quay application tables. Database
  writes are performed by a Quay-image maintenance Job.

## Existing Read-Only Enforcement

Quay already gates read-only behavior on:

```yaml
REGISTRY_STATE: readonly
```

Existing enforcement layers are reused:

- API decorator blocks non-GET requests through `check_readonly`.
- Registry v2 write endpoints are explicitly protected.
- Peewee models derived from `ReadReplicaSupportedModel` block writes.
- Storage layer blocks write operations.
- Workers idle in `Worker.start()` when `REGISTRY_STATE` is readonly.
- `boot.py` refuses to generate a new service key in readonly mode.

No broad changes to read-only enforcement are required. This feature automates
configuration and service key lifecycle around that existing behavior.

**Readonly boot DB-write guards:** `boot.py` currently has two startup paths
that can write before the process starts serving:

1. `sync_database_with_config(app.config)` calls
   `ensure_image_locations()` for `DISTRIBUTED_STORAGE_CONFIG` entries. If a
   location row is missing, it attempts `ImageStorageLocation.insert_many(...)`.
2. `set_region_release(release.SERVICE, release.REGION, release.GIT_HEAD)` uses
   `get_or_create()` on release metadata tables.

In readonly mode these writes are blocked by `ReadReplicaSupportedModel` and can
raise `ReadOnlyModeException`. `conf/init/zz_boot.sh` runs `python boot.py`
directly, and `quay-entrypoint.sh` exits when an init script fails, so these are
startup correctness issues, not cosmetic log noise.

**Fix (MVP):** Guard both write-producing startup paths in `boot.py`:

```python
readonly = app.config.get("REGISTRY_STATE") == "readonly"

if not readonly:
    sync_database_with_config(app.config)
else:
    logger.debug("Registry is in read-only mode, skipping config-to-database sync")

...

if not readonly and release.REGION and release.GIT_HEAD:
    set_region_release(release.SERVICE, release.REGION, release.GIT_HEAD)
```

This avoids all startup DB writes when `REGISTRY_STATE == "readonly"`. Include
tests that readonly boot does not call `ensure_image_locations()` or
`set_region_release()`, including the missing-storage-location case that would
otherwise attempt an insert.

**`boot.py` runs in both `quay-app` and `quay-mirror` containers.** The init
script `conf/init/zz_boot.sh` calls `python boot.py`, and both
`registry-nomigrate` and `repomirror-nomigrate` entrypoint modes execute all
init scripts. This is why the readonly Secret must be mounted in both
containers.

## API and Status Fields

### Spec

```go
type QuayRegistrySpec struct {
    // ReadOnly toggles operator-managed registry read-only mode.
    //
    // nil: operator does not manage REGISTRY_STATE. Existing manual config is
    // preserved.
    //
    // true: operator drives the registry into read-only mode.
    //
    // false: operator drives the registry into normal read-write mode and
    // strips any user-provided REGISTRY_STATE from rendered config.
    ReadOnly *bool `json:"readOnly,omitempty"`
}
```

`*bool` is required. A plain `bool` cannot distinguish "unset" from "explicit
false" and could accidentally revert existing manual readonly deployments
during an operator upgrade.

### Status

```go
type QuayRegistryStatus struct {
    ReadOnlyPhase                ReadOnlyPhase `json:"readOnlyPhase,omitempty"`
    ReadOnlyKeyID                string        `json:"readOnlyKeyID,omitempty"`
    ReadOnlySuppressManualConfig bool          `json:"readOnlySuppressManualConfig,omitempty"`
    ReadOnlyCompatibleImage      string        `json:"readOnlyCompatibleImage,omitempty"`
}
```

`status.readOnlyPhase` is the authoritative state-machine phase.

`status.readOnlyKeyID` stores the public key id (`kid`) for the operator-managed
readonly key. It is public metadata, not secret material. It allows cleanup even
if the Kubernetes Secret containing the private key is deleted.

`status.readOnlySuppressManualConfig` is reserved for future manual readonly
adoption support. It is always `false` in this design and should not be set
by any code path.

`status.readOnlyCompatibleImage` stores the full image reference (including
registry, repository, and tag or digest) of the `quay-app` container that was
running when the phase became non-normal. It is set when transitioning from
`""` to `PreparingKey`, recorded from a **fully rolled-out `quay-app`
Deployment** (not the rendered desired state, and not a Deployment mid-rollout).

**Recording precondition:** Before recording the compatible image, verify
that the live Deployment's rollout is complete:
`ObservedGeneration >= Generation` AND `UpdatedReplicas == Replicas` AND
`AvailableReplicas == Replicas` (matching the existing operator rollout
safety check). If a rollout is in progress (e.g., an image change
was applied in the same or a recent reconcile), defer with
`ReadOnlyDeferred` until the rollout completes. Only then read the image
from the live Deployment's pod template. This ensures the recorded image is
one that pods have actually booted with, not just the desired state of an
in-progress rollout.

Reading from the rolled-out live Deployment (rather than the rendered desired
state) ensures the recorded image is one that has actually been deployed and
is known to work — if `spec.readOnly=true` and an image change are applied
simultaneously, the rendered desired state may contain the new (potentially
incompatible) image, while the live Deployment still has the previous
compatible one. If the live Deployment has no running pods (e.g., scaled to
zero), block with `ReadOnlyDeferred` as with the zero-replicas case. While `readOnlyPhase` is non-normal, if a compatibility check fails
(incompatible custom image without matching annotation), the operator
renders Deployments with this stored image instead of the new one. The field
is cleared when `readOnlyPhase` returns to `""`. `status.currentVersion` is
not sufficient because it does not capture custom image overrides or digests.

Valid phases:

```go
type ReadOnlyPhase string

const (
    ReadOnlyPhaseNormal           ReadOnlyPhase = ""
    ReadOnlyPhasePreparingKey     ReadOnlyPhase = "PreparingKey"
    ReadOnlyPhaseEnteringReadOnly ReadOnlyPhase = "EnteringReadOnly"
    ReadOnlyPhaseReadOnly         ReadOnlyPhase = "ReadOnly"
    ReadOnlyPhaseExitingReadOnly  ReadOnlyPhase = "ExitingReadOnly"
)
```

### Conditions

Add a `ReadOnly` condition type and register it in the operator's known
condition list so `RemoveUnusedConditions` does not prune it.

Reasons:

- `ReadOnlyTransitioning`
- `ReadOnlyActive`
- `ReadOnlyDisabled`
- `ReadOnlyDeferred`
- `ReadOnlyDegraded`
- `ReadOnlyCleanupFailed`
- `OverrideConflict`
- `UnsupportedVersion`
- `ManualMigrationRequired`

Condition behavior:

| State | Status | Reason | Event |
| --- | --- | --- | --- |
| No management, phase `""`, spec nil, suppress=false | absent | - | - |
| PreparingKey | Unknown | ReadOnlyTransitioning | Normal |
| EnteringReadOnly | Unknown | ReadOnlyTransitioning | Normal |
| Stable readonly | True | ReadOnlyActive | Normal |
| ExitingReadOnly | Unknown | ReadOnlyTransitioning | Normal |
| Exit complete | False | ReadOnlyDisabled | Normal |
| Zero app replicas | Unknown | ReadOnlyDeferred | Normal |
| Secret deleted or key invalid (still readonly) | True | ReadOnlyDegraded | Warning |
| Cleanup Job failed (registry is normal, key cleanup blocked) | False | ReadOnlyCleanupFailed | Warning |
| QUAY_OVERRIDE_CONFIG conflict (transition blocked) | False | OverrideConflict | Warning |
| Version too old (transition blocked) | False | UnsupportedVersion | Warning |
| Manual readonly config detected (transition blocked) | False | ManualMigrationRequired | Warning |

**Condition semantics:** `ReadOnly=True` means the registry IS in
**operator-managed** readonly mode. `ReadOnly=False` means it is NOT in
operator-managed readonly mode. Backup controllers can use `status=True` to
know the operator controls readonly state, but should require
`reason=ReadOnlyActive` before starting a backup. `ReadOnlyDegraded` also uses
`status=True` because the registry is still readonly, but it is not a safe
backup-ready signal.

`ReadOnly=False` does **not** guarantee the registry is writable — the
registry may still be in manual readonly (e.g., `ManualMigrationRequired`
is `False` while the registry's source config still has manual readonly
lifecycle fields). The condition reflects operator-managed state only.

- `ReadOnlyDegraded` keeps `True` because the registry IS still in
  operator-managed readonly (just with a degraded key).
- `ReadOnlyCleanupFailed`, `OverrideConflict`, `UnsupportedVersion`, and
  `ManualMigrationRequired` use `False` because the registry is NOT in
  operator-managed readonly mode — these are blocked/error states where the
  transition was refused or exit is incomplete. The actual registry write
  mode may be ambiguous in these states.

Current `updateWithCondition()` emits Warning events for all `ConditionTrue`
statuses. It must be amended so `ReadOnly=True/ReadOnlyActive` emits a Normal
event, and `ReadOnly=True/ReadOnlyDegraded` emits a Warning:

```go
eventType := corev1.EventTypeNormal
if cstatus == metav1.ConditionTrue {
    if ctype == v1.ConditionTypeReadOnly &&
        reason == v1.ConditionReasonReadOnlyActive {
        eventType = corev1.EventTypeNormal
    } else {
        eventType = corev1.EventTypeWarning
    }
}
```

`ReadOnly=False` with Warning reasons (`OverrideConflict`,
`UnsupportedVersion`, `ManualMigrationRequired`, `ReadOnlyCleanupFailed`)
uses the default Normal event type (since `ConditionFalse != ConditionTrue`),
but each of these should emit a Warning event. Add explicit handling:

```go
if ctype == v1.ConditionTypeReadOnly && isWarningReason(reason) {
    eventType = corev1.EventTypeWarning
}

func isWarningReason(reason v1.ConditionReason) bool {
    switch reason {
    case v1.ConditionReasonReadOnlyDegraded,
         v1.ConditionReasonReadOnlyCleanupFailed,
         v1.ConditionReasonOverrideConflict,
         v1.ConditionReasonUnsupportedVersion,
         v1.ConditionReasonManualMigrationRequired:
        return true
    }
    return false
}
```

`status.readOnlyPhase` remains authoritative when the condition reason is an
abnormal overlay such as `OverrideConflict` or `ReadOnlyCleanupFailed`.

## Service Key Model

### Why a Durable Secret Is Required

`boot.py` writes the instance service key files to `CONF_DIR`, which resolves to
the container filesystem, not the mounted `/conf/stack` config Secret. Files
written there are lost on pod replacement.

The Quay database stores public JWK material only. It must not store private
JWK material because the keyserver returns `key.jwk` directly. Therefore the
operator persists private key material in a Kubernetes Secret instead.

### Secret

Name:

```text
<quayregistry-name>-readonly-service-key
```

Data:

```text
quay-readonly.kid
quay-readonly.pem
```

Required properties:

- ownerReference to the `QuayRegistry`, except where finalizer cleanup needs a
  different object ownership model for Jobs.
- labels:
  - `app.kubernetes.io/managed-by: quay-operator`
  - `quay-operator/readonly-key: "true"`
- `immutable: true`
- mounted as a **separate** Secret volume named `readonly-service-key` with
  `defaultMode: 0444`. This volume is independent of the existing `config`
  projected volume (which combines `quay-config-secret` and `quay-config-tls`
  at `/conf/stack`). The readonly Secret has a different lifecycle — it is
  absent in non-readonly phases and should not be mixed into the config
  projection.

`defaultMode` is a volume property, not a Secret object property.

**File mode rationale:** Use `0444` (world-readable) rather than `0440`
(group-readable). Quay runs as a non-root user, and operators/admins can
override the container `securityContext` (e.g., `runAsUser`, `runAsGroup`,
`fsGroup`). If the process UID is not in the file's owning group,
`0440` would prevent Quay from reading the PEM and cause `boot.py` to fail.
Since the Secret contains a public/private key pair that is already RBAC-
protected at the Kubernetes API level and only mounted into Quay containers,
`0444` is acceptable. The private key is not world-readable on the host
filesystem — it exists only in the container's tmpfs-backed Secret volume.

The Secret is immutable to prevent in-place mutation of key material while Quay
processes cache `instance_keys.local_key_id` and `instance_keys.local_private_key`.
If an existing immutable Secret does not match `status.readOnlyKeyID`, the
operator must not attempt unsafe repair. Condition handling is phase-scoped:

- During `PreparingKey`, the registry is still normal/read-write. Set
  `ReadOnly=Unknown/ReadOnlyTransitioning` with a clear message and do not
  advance. Do not use `ReadOnlyDegraded` because `ReadOnly=True` would
  incorrectly signal that the registry is already operator-managed readonly.

### Mount Targets

Mount the Secret only into containers that run Quay boot and need JWT signing:

| Workload | Container | Mount |
| --- | --- | --- |
| `quay-app` Deployment | `quay-app` | yes |
| `quay-mirror` Deployment | `quay-mirror` | yes, if mirror is managed |
| `quay-mirror` init container | init | no |
| config editor | any | no |
| migration Job | any | no |
| cleanup Job | cleanup | no private key needed |

Paths injected into rendered config:

```yaml
INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES: true
INSTANCE_SERVICE_KEY_EXPIRATION: 43200
INSTANCE_SERVICE_KEY_KID_LOCATION: /conf/readonly/quay-readonly.kid
INSTANCE_SERVICE_KEY_LOCATION: /conf/readonly/quay-readonly.pem
```

`INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` is an internal opt-in guard for the
new normal-mode import path. It defaults to `false` in `config.py` and must be
added to `util/config/schema.py` and `INTERNAL_ONLY_PROPERTIES`, matching the
existing instance-service-key settings. It is operator-owned and must not be
documented as an administrator-facing config knob.

Quay must not infer operator-managed import merely from the presence of key
files: existing normal deployments already have `/conf/stack/quay.kid` and
`/conf/stack/quay.pem` after first boot, and current normal-mode behavior is to
generate and overwrite the instance key files. Without this guard, a normal
restart could incorrectly mark an ordinary instance key as operator-managed
readonly material.

`INSTANCE_SERVICE_KEY_EXPIRATION` is in **minutes** (used as
`timedelta(minutes=...)` in `boot.py`). `43200` minutes = 30 days. This
bounded expiration ensures orphaned keys self-expire if cleanup fails (see
Finalizer Cleanup). The `ServiceKeyWorker` refreshes the expiration normally
while the registry runs in read-write mode.

**Maximum readonly window:** The 30-day bounded expiration imposes an
operational limit: the registry can remain in readonly mode for at most 30
days from the last key refresh. In readonly mode, the `ServiceKeyWorker` is
idle (`Worker.start()` skips all operations), so the expiration is not
refreshed. After 30 days, the key expires and `boot.py` will fail
verification on the next pod restart, causing `CrashLoopBackOff`.

Mitigations:

- **Pre-expiry warning:** On each reconcile while `readOnlyPhase == "ReadOnly"`,
  the operator calls the key status endpoint
  `GET /keys/services/quay/keys/<kid>/status` and reads `expiration_date`
  from the response. If the key is approaching expiration
  (`now < expiration_date AND expiration_date <= now + 7 days`), set
  `ReadOnly=True/ReadOnlyDegraded` with a message like "Service key expires
  in N days; exit readonly mode or the registry will become unavailable."
  Emit a Warning event. Already-expired keys (`expiration_date <= now`) are
  not covered by this predicate.
  This uses the actual DB expiration, not a computed estimate — the
  `ServiceKeyWorker` may have refreshed the key shortly before readonly was
  entered, making the true deadline later than `Secret creation + 30 days`.
- **Documentation:** State that operator-managed readonly mode supports
  continuous operation for up to 30 days. For longer maintenance windows,
  the admin should exit readonly, let the worker refresh the key, then
  re-enter. Manual DB updates (`UPDATE servicekey SET expiration_date ...`)
  are not supported for operator-managed readonly keys and must not be used
  to extend the window — the operator tracks key state via the `/status`
  endpoint and `metadata.created_by` marker, and manual edits can leave
  these out of sync.
- The E2E test "Readonly survives longer than 120 minutes" validates
  short-term operation. Add a unit test that verifies the pre-expiry
  warning fires when the key is within 7 days of expiration.

**Stable ReadOnly key health check:** On each reconcile while
`readOnlyPhase == "ReadOnly"`, the operator runs the same two-step
verification used in `PreparingKey`:

Step 1 — raw GET `GET /keys/services/quay/keys/<kid>`:
- 200: key exists, is approved, and is alive. Continue to step 2.
- 404/409/403: key is missing, unapproved, or expired. Set
  `ReadOnly=True/ReadOnlyDegraded` with a message describing the failure.
  This is critical because Quay's `InstanceKeys` cache uses
  `list_service_keys()` with `approved_only=True, alive_only=True` — an
  expired or unapproved key is excluded from the cache, breaking JWT
  verification at runtime or on the next pod restart.

Step 2 — `/status` endpoint `GET /keys/services/quay/keys/<kid>/status`:
- Check `operator_managed == true`. If false: `ReadOnlyDegraded`.
- Check pre-expiry: if `now < expiration_date <= now + 7 days`, set
  `ReadOnlyDegraded` with a warning that the key is approaching expiration.
- Check PEM/JWK consistency (Secret PEM-derived `n`/`e` vs raw GET JWK).
  If mismatch: `ReadOnlyDegraded`.

None of these force an automatic exit — the admin must decide whether to
exit or remediate. The degraded condition provides the signal.

## Quay Code Changes

### Schema Changes

No Quay database schema migration is required. The design reuses the existing
`ServiceKey.metadata` JSONField to store
`{"created_by": "quay-operator-readonly"}`, so there is no Alembic migration
and no change to `data/database.py` table definitions.

The only Quay-side schema/default change is config schema support for
`INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES`:

- Add `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES = False` to `DefaultConfig`.
- Add the key to `util/config/schema.py` with boolean type.
- Add the key to `INTERNAL_ONLY_PROPERTIES`, matching the existing
  instance-service-key config fields.
- Add/update schema tests so `test_ensure_schema_defines_all_fields` passes and
  the field remains internal-only.

The operator still requires CRD/OpenAPI schema changes for `spec.readOnly` and
the readonly status fields described above.

### `boot.py` Decision Tree

When mounted key files are supported (bounded expiration model):

```text
setup_instance_service_key()
  |
  |-- REGISTRY_STATE == "readonly"
  |     -> verify only; no writes
  |
  |-- REGISTRY_STATE != "readonly"
  |   and INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES == true
  |   and configured key files exist
  |     -> import key from files; ensure DB row with
  |        expiration_date = now + INSTANCE_SERVICE_KEY_EXPIRATION minutes
  |
  |-- REGISTRY_STATE != "readonly"
  |   and INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES == true
  |   and configured key files missing or invalid
  |     -> fail boot with clear error; do not generate a replacement key
  |
  `-- REGISTRY_STATE != "readonly" and import flag false
        -> current TTL behavior (generate new key, write to CONF_DIR)
```

The import branch is used only when the operator has rendered
`INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES: true` together with the readonly Secret
mount paths. It must not trigger only because the default instance key files
exist. When the flag is true, missing or invalid files are an operator/Secret
mount failure and must not fall back to generating a replacement key. The
mounted-key-file path uses the same
`INSTANCE_SERVICE_KEY_EXPIRATION` config (in minutes) as the existing TTL path.
The operator sets this to `43200` (30 days). The `ServiceKeyWorker` refreshes
the expiration normally.

### Import From Files

Add `_import_service_key_from_files()`:

- Read `kid` from `.kid`.
- Load private key from `.pem`.
- Derive public JWK.
- Verify computed JWK thumbprint equals the `.kid` value.
- Store public JWK only, including `kid`.
- Use `name="operator-managed-readonly"` and `rotation_duration=None` when
  creating the `ServiceKey` row. The descriptive name makes operator-managed
  keys identifiable in the service key listing UI and API responses.
- Set `metadata = {"created_by": "quay-operator-readonly"}` on the DB row.
  The `ServiceKey.metadata` column is a `JSONField` that accepts arbitrary
  key-value data. This marker enables the cleanup CLI to distinguish
  operator-managed keys from manually created ones, and refuse to expire
  keys it did not create.
- If a DB row already exists:
  - require `service == app.config["INSTANCE_SERVICE_KEY_SERVICE"]` (default
    `"quay"`). Use the config value for consistency with the existing
    `generate_key()` call path in boot.py, not a hardcoded string. The
    operator and `manage_servicekey.py` CLI may hardcode `"quay"` since they
    are quay-specific tools.
  - require DB JWK `n/e` match the PEM-derived JWK
  - check `metadata.created_by`. First, normalize metadata defensively:
    if `key.metadata` is not a `dict` (e.g., `None`, a JSON string, or
    other malformed legacy data), replace it with `{}` before proceeding.
    This matches the same `isinstance(key.metadata, dict)` guard used in
    the `/status` endpoint:
    - if `created_by` is absent or empty (`None`, `""`, or key missing
      from dict): backfill it:
      `key.metadata = key.metadata if isinstance(key.metadata, dict) else {}`
      `key.metadata.update({"created_by": "quay-operator-readonly"})`
      and save. This covers keys from earlier boot cycles or previous
      operator versions that did not set the marker.
    - if already set to `"quay-operator-readonly"`: no action needed.
    - if set to a **different** value (e.g., `"manual"`, `"admin"`): **fail
      with an error.** A key explicitly marked as non-operator-managed must
      not be silently claimed. This prevents the operator from adopting a
      manually provisioned key that cleanup would later refuse to expire,
      and avoids overwriting an admin's intentional marker.
  - If the existing row is operator-managed and its public JWK matches the
    mounted PEM-derived JWK, refresh `expiration_date` to
    `now + INSTANCE_SERVICE_KEY_EXPIRATION` even if the current DB value is
    already expired. This is allowed only in the normal-mode import path where
    Quay can write to the DB. It is required for re-entry after a cleanup Job
    has shortened the key lifetime or after a long interrupted exit.

  **GC ordering requirement:** The import implementation must look up the
  existing row before any model path that triggers `_gc_expired(service)`.
  Today `create_service_key()` calls `_gc_expired()` before inserting, and GC
  physically deletes stale expired keys for the service. If import calls the
  create path first, a stale matching operator-managed row (or a conflicting
  explicitly non-operator row) can be deleted before import has a chance to
  refresh or reject it. Only after the direct lookup confirms that no row
  exists should import create a new row; concurrent create races still use the
  `IntegrityError` recovery flow below.

  **Lookup must use `kid` only, not `kid + service`.** `ServiceKey.kid` is
  globally unique (`unique=True` in `data/database.py`). A lookup by
  `WHERE kid=X AND service=Y` would miss a row with `kid=X, service=Z`, and
  the subsequent create would hit an `IntegrityError` on the unique constraint
  instead of producing a clean "wrong service" rejection. The correct pattern:

  ```python
  try:
      key = ServiceKey.select().where(ServiceKey.kid == kid).get()
  except ServiceKey.DoesNotExist:
      key = None
  ```

  Then validate `key.service` explicitly against
  `app.config["INSTANCE_SERVICE_KEY_SERVICE"]` as a separate check. This
  produces a clear, actionable error message for wrong-service keys.

  **Ownership rule summary:** There are two paths that check key ownership,
  with intentionally different behavior:

  The registry is NOT readonly during import (`PreparingKey` / normal boot).
  `boot.py` can write to the DB. If `created_by` is absent/empty, backfill
  the marker — the key was likely created by a prior operator version.
  If `created_by` has a conflicting value, fail — the key belongs to
  someone else. A matching operator-managed row may be expired; import
  refreshes it rather than rejecting it.
- **Auto-approve the key.** The existing `generate_key()` path in
  `util/generatepresharedkey.py:20` auto-approves via
  `approve_service_key(kid, ServiceKeyApprovalType.AUTOMATIC)`. The import path
  must do the same for both new rows and existing rows: if the key has no
  approval, call `approve_service_key()` with
  `ServiceKeyApprovalType.AUTOMATIC`. Without approval, the raw GET keyserver
  endpoint returns 409, and PreparingKey verification will never advance past
  step 1. This is a required step, not optional. **Idempotency:**
  `approve_service_key()` raises `ServiceKeyAlreadyApproved` if an approval
  record already exists. The import path must check `key.approval is not None`
  before calling; if already approved, skip the approval step and log at info
  level. This handles retry scenarios and partial import recovery.
  **Concurrent approval race:** Both `quay-app` and `quay-mirror` run
  `boot.py` and can import/approve the same key simultaneously. Even after
  the pre-check, a concurrent caller can approve the key between the check
  and the `approve_service_key()` call. The import path must catch
  `ServiceKeyAlreadyApproved`, re-read the key, verify the approval exists,
  and treat it as success. This is the same pattern as the `IntegrityError`
  recovery for concurrent creates below.
- If create races with another pod:
  - catch `IntegrityError`
  - re-read the key
  - perform the same validation before continuing

The validation should live in a shared helper used by import, race recovery, and
readonly verification.

### Readonly Verification

Harden `_verify_service_key()`:

- Strip whitespace from `.kid`.
- Call `get_service_key(kid,
  service=app.config["INSTANCE_SERVICE_KEY_SERVICE"],
  approved_only=True, alive_only=True)`.
- Require PEM file to exist.
- Load PEM.
- Derive public JWK.
- Verify computed thumbprint equals `.kid`.
- Verify derived `n/e` equal DB JWK `n/e`.
- Return `None` instead of raising `AssertionError` for missing files.

**Behavioral change note:** The current `_verify_service_key()` uses
`approved_only=False` and does not pass a `service` parameter. This change
to `approved_only=True` with the configured service name is intentional:
only approved, service-matching keys should grant readonly access. This means
if a key becomes unapproved (e.g., admin revokes approval via API), readonly
boot would fail where it previously succeeded. The import path mitigates
this by auto-approving the key during normal-mode import.

### Service Key Worker

No changes needed. The `ServiceKeyWorker` refreshes the expiration normally
using the configured `INSTANCE_SERVICE_KEY_EXPIRATION` value (43200 minutes for
operator-managed readonly). The bounded expiration is refreshed on the existing
schedule. In readonly mode, the worker is idle (stopped by `Worker.start()`),
but the 30-day window provides sufficient runway.

### Config Schema

No DB schema changes are needed. `INSTANCE_SERVICE_KEY_EXPIRATION` remains
`"type": "number"` (its existing config schema type). The operator injects
`43200` (a valid number). Null is not used.

One config schema/default change is required for the import guard:
`INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` must be added as a boolean config key,
defaulting to `False`, and listed in `INTERNAL_ONLY_PROPERTIES`.

### Maintenance CLI

Add:

```text
/quay-registry/tools/manage_servicekey.py
```

Required command:

```text
expire --kid <kid> --grace-seconds <seconds>
```

**DB lookup requirement:** The CLI must NOT use the existing
`get_service_key()` or `set_key_expiration()` helpers, because both go
through `_list_service_keys_query()` which applies stale-expired filtering
(`data/model/service_keys.py`). That filter silently drops keys that
expired more than `EXPIRED_SERVICE_KEY_TTL_SEC` (7 days) ago. If the cleanup
Job runs late (e.g., prolonged `ExitingReadOnly` with Job failures), the
existing helpers would return `ServiceKeyDoesNotExist`, collapsing "wrong
service" or "non-operator stale row" into "missing key → exit 0" and
bypassing the unconditional ownership check.

Instead, the CLI must use a direct lookup that bypasses stale-expired
filtering — the same pattern as the `/status` endpoint's
`get_service_key_for_status()`:

```python
key = (ServiceKey
    .select()
    .where(ServiceKey.kid == kid)
    .get())
```

Then validate `service` and `metadata.created_by` explicitly before deciding
whether to no-op or update expiration. Similarly, the expiration update must
use a direct `key.save()` on the looked-up row, not `set_key_expiration()`.

Behavior (in this order):

1. Look up key by `kid` using direct query. If row does not exist: exit 0.
2. Validate `key.service == "quay"`. If wrong service: fail.
3. Validate `metadata.created_by == "quay-operator-readonly"`. If not
   operator-managed: **always fail**, regardless of existing expiration value.
   This is unconditional — the CLI never expires a non-operator key. A future
   `--force` flag may override this for admin use.
4. Check idempotency: if key is already expired or already expiring within the
   requested grace window: exit 0.
5. Set `expiration_date = now + grace`.

Service and ownership validation (steps 2-3) must run before the idempotent
no-op check (step 4). Otherwise an expired non-operator key or a key for a
different service would hit "already expired → exit 0" and bypass the
ownership check.

Do not require a `delete` command for operator use. If added for admin/test
use, require an explicit confirmation flag.

The cleanup command must be able to run even when the cluster's rendered Quay
config still contains `REGISTRY_STATE: readonly`. The cleanup Job must therefore
set `QUAY_OVERRIDE_CONFIG` with `REGISTRY_STATE: normal`:

```json
QUAY_OVERRIDE_CONFIG={"REGISTRY_STATE":"normal"}
```

If the user has an allowed inline `QUAY_OVERRIDE_CONFIG` that does not contain
any blocked readonly lifecycle key (see Override Conflict Detection), merge it
into the cleanup Job override and let the operator's `REGISTRY_STATE: normal`
value win:

```json
QUAY_OVERRIDE_CONFIG={"USER_ALLOWED_KEY":"value","REGISTRY_STATE":"normal"}
```

Do not drop allowed user override keys in the cleanup Job. `valueFrom`-based
`QUAY_OVERRIDE_CONFIG` remains blocked during readonly transitions because it
cannot be inspected safely.

## Operator Reconcile Design

### Pre-Inflate Key Secret Creation

The key Secret must not be generated inside generic render/apply code. The
operator uses server-side apply (SSA) via `client.Apply` for all non-Job
objects in `createOrUpdateObject()`. Once a Secret has `Immutable: true`, SSA
rejects any patch that touches its `data` fields — even if unchanged. The
readonly Secret must therefore be created and managed by
`ensureReadOnlyKeySecret()` directly (using `r.Create()`), and must NOT be
included in the `Inflate()` output or the `deploymentObjects` slice that
flows through `createOrUpdateObject()`. Only the volume/mount reference on
the Deployment is rendered by `Inflate()`.

Run `ensureReadOnlyKeySecret()` before `Inflate()`. Behavior is
phase-scoped:

**PreparingKey:**
1. If Secret exists:
   - verify required keys are present
   - verify `.kid` matches `status.readOnlyKeyID` if status is set
   - if status is empty, set `status.readOnlyKeyID` from Secret data
2. If Secret does not exist:
   - generate RSA 2048 key pair
   - compute RFC 7638 thumbprint
   - create immutable Secret
   - set `status.readOnlyKeyID`
3. Persist status and requeue if `readOnlyKeyID` changed.

**EnteringReadOnly / ReadOnly:**
- If Secret exists and matches `status.readOnlyKeyID`: no action.
- If Secret is missing, has wrong data, or does not match
  `status.readOnlyKeyID`: set `ReadOnly=True/ReadOnlyDegraded` (if in
  `ReadOnly`) or `ReadOnly=Unknown/ReadOnlyTransitioning` (if in
  `EnteringReadOnly`). Do NOT generate a new key — Quay cannot import a
  new key while readonly, and pods may cache the existing key. Do NOT
  change `status.readOnlyKeyID`.

**ExitingReadOnly:**
- The Secret may or may not exist. The cleanup Job uses
  `status.readOnlyKeyID` (not the Secret) to identify the DB key to expire.
  Do not require the Secret for cleanup to proceed.

Only after this does `Inflate()` render config and deployment mounts.

### Reconcile Flow

```text
Reconcile()
  |
  |-- targetPhase := selectReadOnlyPhase()
  |     computes target phase from spec + current status; does NOT write status
  |
  |-- ensureReadOnlyKeySecret()
  |     phase-scoped: create in PreparingKey, verify in others
  |
  |-- objects := Inflate(ctx, ...)
  |     renders config and volumes based on targetPhase
  |
  |-- Apply(objects)
  |     applies objects via SSA
  |
  `-- advanceReadOnlyPhase(targetPhase)
        checks rollout/Job status, writes readOnlyPhase to CR status,
        sets conditions, returns RequeueAfter
```

**`selectReadOnlyPhase()` semantics:**

```go
func (r *QuayRegistryReconciler) selectReadOnlyPhase(
    ctx context.Context,
    quay *v1.QuayRegistry,
) (ReadOnlyPhase, error)
```

This function computes the target phase from `spec.readOnly` and the current
`status.readOnlyPhase`. It does NOT modify the CR status or write any
conditions. It returns the phase that `Inflate()` should render for and that
`advanceReadOnlyPhase()` should attempt to reach.

The separation is intentional: `Inflate()` and `Apply()` must succeed before
the phase is written to status, keeping phase transitions atomic with the
object mutations they represent. If `Apply()` fails, the phase is not
advanced, and the next reconcile re-derives the same target.

`advanceReadOnlyPhase()` receives the target phase and is responsible for:
1. Checking rollout/Job preconditions for the target phase transition
2. Writing `status.readOnlyPhase` when preconditions are met
3. Setting/updating the `ReadOnly` condition
4. Returning the appropriate `RequeueAfter` interval

Stable `ReadOnly` and stable normal mode must continue through `Inflate()` so
unrelated changes, health checks, secret rotations, and image changes still
reconcile.

**Periodic reconcile during stable ReadOnly:** Readonly phase handlers must not
depend on watched-object events or status-only updates. The main controller
uses `GenerationChangedPredicate`, so status-only
updates do not trigger reconciliation, and readonly-specific early returns can
bypass the normal end-of-reconcile `RequeueAfter`. Without an explicit periodic
requeue from readonly paths, pre-expiry warnings, key health checks, the
`quay_operator_readonly_active` metric, and image compatibility checks may not
run until an unrelated spec change occurs.

`advanceReadOnlyPhase()` MUST return `RequeueAfter: 5 * time.Minute` when
`readOnlyPhase == "ReadOnly"` to ensure periodic health monitoring. This
5-minute interval balances observability (pre-expiry warnings, degraded key
detection) against reconcile cost (one reconcile per 5 minutes per CR is
negligible). The same applies during `ExitingReadOnly` while waiting for a
cleanup Job to complete — use `RequeueAfter: 30 * time.Second` to poll Job
status, since Job status changes do not increment `metadata.generation` and
will not trigger a reconcile via `GenerationChangedPredicate`.

**Pre-expiry and health check placement:** All key health checks (pre-expiry
warning) run inside `advanceReadOnlyPhase()`
when `currentPhase == ReadOnly`. This function already handles phase-specific
logic and condition updates, making it the natural location for stable-phase
health monitoring.

### Phase Advancement Checks

For the `quay-app` deployment:

- `ObservedGeneration >= Generation`
- `UpdatedReplicas == desired`
- desired replicas must be greater than 0, otherwise set
  `ReadOnlyDeferred` and do not advance

`desired` means:

- If HPA is unmanaged: `Deployment.spec.replicas` after middleware/defaulting.
- If HPA is managed and phase is `EnteringReadOnly` or `ExitingReadOnly`: the
  readonly pinned count written to the rendered HPA (`minReplicas ==
  maxReplicas == pinned count`). The operator must not rely on an unset
  Deployment replica field while HPA owns scaling.
- If HPA is managed and phase is not a Recreate transition: use the live HPA
  desired/current deployment counts for health observation only; do not advance
  a readonly transition from those phases.

Availability:

- `PreparingKey`: `AvailableReplicas >= 1`, proving at least one pod booted and
  imported the key. **Additionally, verify key import by querying
  `GET /keys/services/quay/keys/<kid>` on the Quay service ClusterIP.** Pod availability only proves the health probe passed, not that
  `boot.py` successfully created the DB row. The keyserver check is read-only
  and does not violate the constraint against direct DB mutation.

  **Keyserver caveat:** The current `get_service_key` endpoint
  (`endpoints/keyserver/__init__.py`) has two problems:
  1. It ignores the `service` path parameter — fetches by `kid` only.
  2. It returns `jsonify(key.jwk)` — the raw JWK dict containing
     public RSA key fields (`kty`, `n`, `e`). The JWK may or may not
     include `kid` depending on how the key was created: Authlib's
     `as_dict()` includes it, but externally submitted keys via PUT are
     stored as-is and may omit it. The response does not include `service`,
     `metadata`, or any other `ServiceKey` model fields, and must never
     include private JWK fields (`d`, `p`, `q`, `dp`, `dq`, `qi`). The
     response is NOT wrapped in an envelope — the JWK fields are at the
     top level of the JSON object.

  Client-side `service` verification is therefore impossible with the current
  response format. **For v1, add a new operator-facing endpoint** rather than
  changing the existing GET response shape. The current GET returns the raw
  JWK object (`jsonify(key.jwk)`), and any existing client expecting `kty`,
  `n`, `e` at the top level would break if wrapped in an envelope.

  1. **Keep the existing GET endpoint unchanged.** Fix only the `service`
     filtering: pass `service` to `model.get_service_key()`, consistent with
     PUT/DELETE. The response remains `jsonify(key.jwk)`.

  2. **Add a new endpoint** for operator verification:
     ```
     GET /services/<service>/keys/<kid>/status
     ```
     Response:
     ```python
     exp = None
     if key.expiration_date is not None:
         exp = key.expiration_date.strftime("%Y-%m-%dT%H:%M:%SZ")
     resp = jsonify({
         "kid": key.kid,
         "service": key.service,
         "operator_managed": (key.metadata if isinstance(key.metadata, dict) else {}).get("created_by") == "quay-operator-readonly",
         "expiration_date": exp,
     })
     ```
     **Time format:** `expiration_date` is RFC 3339 UTC with explicit `Z`
     suffix (e.g., `"2026-09-16T03:54:00Z"`). Quay stores naive UTC
     datetimes in the database; Python's `isoformat()` omits the timezone
     marker, which the Go operator would parse incorrectly or reject when
     using `time.Parse(time.RFC3339, ...)`. Use `strftime` with explicit
     `Z` to produce a deterministic RFC 3339 string.

     **Error semantics:** Unlike the raw GET endpoint (which returns 409 for
     unapproved and 403 for expired keys), `/status` always returns 200 with
     the full status payload if the key exists for the given service — even
     if the key is expired or unapproved. This allows the operator to read
     `expiration_date` for pre-expiry warnings on keys approaching expiration
     and to inspect `operator_managed` regardless of approval state. Return
     404 only if the key does not exist for the given service.

     **Stale-expired key filtering:** The existing `_list_service_keys_query()`
     in `data/model/service_keys.py` filters out keys that expired more
     than `EXPIRED_SERVICE_KEY_TTL_SEC` (default: 7 days) ago. This means
     the `/status` endpoint will return 404 for keys that have been expired
     for longer than 7 days, even though the DB row still exists. The
     `/status` endpoint must use a dedicated model lookup that bypasses
     stale-expired filtering. The keyserver blueprint uses `pre_oci_model`
     (imported as `model` in `endpoints/keyserver/__init__.py`) which
     delegates through `models_interface.py`. Route the new lookup through
     the same interface chain:

     1. `endpoints/keyserver/models_interface.py` — add abstract method:
        `get_service_key_for_status(kid, service)`
     2. `endpoints/keyserver/models_pre_oci.py` — implement by delegating
        to `data_model.get_service_key_for_status(kid, service)`
     3. `data/model/service_keys.py` — add:
        ```python
        def get_service_key_for_status(kid, service):
            try:
                return (ServiceKey
                    .select()
                    .where(ServiceKey.kid == kid, ServiceKey.service == service)
                    .get())
            except ServiceKey.DoesNotExist:
                raise ServiceKeyDoesNotExist
        ```
        No `ServiceKeyApproval` JOIN is needed — the `/status` response does
        not include approval state, only `kid`, `service`, `operator_managed`,
        and `expiration_date`.

     The `/status` endpoint calls `model.get_service_key_for_status(kid,
     service)`, keeping database-query logic out of the endpoint and
     following the existing keyserver model interface pattern.

     This ensures the operator can read the status of an operator-managed
     key even if it has been expired for longer than 7 days, **as long as
     the DB row still exists.** The `_gc_expired()` function in
     `_gc_expired()` in `data/model/service_keys.py` physically deletes stale expired keys
     during `create_service_key()`, `replace_service_key()`, and
     `delete_service_key()` calls. In readonly mode, none of these are
     called (the model layer blocks writes), so the row is safe. After
     exiting readonly, a subsequent key creation for the same service could
     GC the row. If `/status` returns 404 after GC, the operator treats
     this as idempotent cleanup success — the key no longer exists and
     does not need expiration.

     This endpoint returns metadata the operator needs (ownership, expiry)
     without changing the existing JWK response contract. It is still
     unauthenticated (consistent with the keyserver blueprint) but exposes
     only narrow, non-sensitive fields. Including `expiration_date` enables
     the pre-expiry warning (see Maximum Readonly Window).

     **Security decision (accepted low risk):** The `/status` endpoint is
     intentionally public (unauthenticated). This is acceptable because:
     (1) the `kid` is already referenced in operator-controlled config and
     is not secret; (2) the response contains no private key material, no
     config contents, and no user data; (3) `operator_managed` and
     `expiration_date` are operational metadata with no security
     sensitivity; (4) the existing keyserver endpoints are already
     unauthenticated and expose the full public JWK; (5) the endpoint is
     only reachable in-cluster via ClusterIP or via the external route's
     `/keys` path, which is already public.

     **Accepted risk:** The endpoint does expose incremental metadata
     disclosure — key existence, operator-managed status, and expiration
     timing — to anyone who knows or guesses a `kid`. This is low risk
     because `kid` values are opaque RFC 7638 thumbprints (not guessable),
     the information has no direct exploit value, and the existing keyserver
     already confirms key existence via the raw GET endpoint. Tests must
     ensure `/status` never returns full `metadata`, private JWK fields
     (`d`, `p`, `q`, `dp`, `dq`, `qi`), or any fields beyond `kid`,
     `service`, `operator_managed`, and `expiration_date`.

  Both changes are scoped to the keyserver blueprint.

  **URL construction:** The operator calls the `quay-app` ClusterIP Service
  (`<name>-quay-app`) in the QuayRegistry's namespace. The service exposes
  port 80 (HTTP → targetPort 8080) and port 443 (HTTPS → targetPort 8443).
  Use **HTTP on port 80** for in-cluster verification — this is an internal
  call that does not leave the cluster, and avoids TLS certificate
  verification complexity (the service TLS cert is for the external route
  hostname, not the internal service name). The full URL is:
  ```
  http://<name>-quay-app.<namespace>.svc/keys/services/quay/keys/<kid>
  ```
  Use the short-form `<name>.<namespace>.svc` rather than
  `<name>.<namespace>.svc.cluster.local` — the cluster DNS suffix is
  configurable and may not be `cluster.local` in all environments.
  **Timeout:** 10 seconds per request, 3 retries with 5-second backoff.
  Verification failures are retried on the next reconcile (1-minute
  requeue), so aggressive retries within a single reconcile are unnecessary.

  The operator verification logic (all paths are full HTTP paths including
  the blueprint prefix `/keys`; the Flask route is
  `/services/<service>/keys/<kid>` registered at `url_prefix="/keys"` in
  `web.py`):

  Step 1 — existence/approval check via the existing endpoint:
  1. Call `GET /keys/services/quay/keys/<kid>`.
  2. If 404: key not imported, do not advance.
  3. If 409: key not approved, do not advance.
  4. If 403: key expired, do not advance.
  5. If 200: key exists, is approved, and is alive. Continue to step 2.

  Step 2 — ownership and consistency check via the status endpoint:
  1. Call `GET /keys/services/quay/keys/<kid>/status`.
  2. Require `operator_managed == true`. If false, the key was not created by
     the operator — set `ReadOnly=Unknown/ReadOnlyTransitioning` with a
     message describing the failure. Do not advance.
  3. Load the PEM from the readonly Secret, derive the public JWK, compute
     the RFC 7638 thumbprint. Verify:
     - Computed `kid` matches the Secret's `.kid` value.
     - Derived `n` and `e` match the JWK from the step-1 response.
     Note: use `kid` from the Secret's `.kid` file and from the `/status`
     endpoint, not from the raw GET JWK body — the raw JWK may omit `kid`
     for externally submitted keys.
  4. If any mismatch: set `ReadOnly=Unknown/ReadOnlyTransitioning` with a
     message describing the mismatch. Do not advance.

  **Condition status during PreparingKey failures:** Use `Unknown` (not
  `True/ReadOnlyDegraded`) because the registry is NOT in readonly mode
  during `PreparingKey` — `REGISTRY_STATE: readonly` has not been injected
  yet. `ReadOnlyDegraded` with `status=True` is reserved for states where
  the registry IS readonly but has a degraded key (e.g., Secret deleted
  during stable `ReadOnly`).

  5. All checks pass: advance to `EnteringReadOnly`.
- `EnteringReadOnly`: `AvailableReplicas == desired`
- `ExitingReadOnly`: `AvailableReplicas == desired`

For `quay-mirror`, check only if `ComponentIsManaged(..., ComponentMirror)` is
true.

### Readonly Secret Volume Mount

The readonly Secret volume and mount are present on `quay-app` (and managed
`quay-mirror`) Deployments only during phases that need key material:

| Phase | Volume Mount | Rationale |
| --- | --- | --- |
| Normal (`""`) | No | No readonly key exists |
| PreparingKey | Yes | Pods import key from mounted files |
| EnteringReadOnly | Yes | Pods use key for JWT signing |
| ReadOnly | Yes | Steady-state operation |
| ExitingReadOnly | No | Config key paths removed; key being expired |

Like the deployment strategy, the volume mount is **derived from the target
phase returned by `selectReadOnlyPhase()` on every reconcile**, not from
`status.readOnlyPhase`. This distinction matters during abort sequences:
when `spec.readOnly` changes to `false` during `PreparingKey`, the target
phase is `""` (normal) even though `status.readOnlyPhase` is still
`PreparingKey`. `Inflate()` must render without the mount so pods roll off
the key files. All rendering decisions — config injection, volume mounts,
deployment strategy, `terminationGracePeriodSeconds` — use the target phase.
The status phase is written only by `advanceReadOnlyPhase()` after Apply
succeeds. Do not delete the existing readonly Secret during `ExitingReadOnly` — it
remains managed by direct Secret lifecycle code (`ensureReadOnlyKeySecret()`
/ `r.Delete()`), not by `Inflate()`/SSA. The Secret is not mounted into the
Deployment during `ExitingReadOnly` but must persist until the cleanup Job
completes.

**Rendering mechanism:** The volume and mount are injected by post-processing
the Kustomize output inside `Inflate()`, not via a Kustomize overlay. After
`generate()` produces the base object list:

1. Find the `quay-app` Deployment in the rendered object list.
2. If the target phase (from `selectReadOnlyPhase()`) requires the mount
   (see table above), append
   the `readonly-service-key` Volume (Secret source, `defaultMode: 0444`) to
   `spec.template.spec.volumes` and append the corresponding VolumeMount
   (`mountPath: /conf/readonly`, `readOnly: true`) to the `quay-app` container.
3. Repeat for the `quay-mirror` Deployment if mirror is managed.
4. If the phase does not require the mount, do not add the volume or mount.

This approach avoids overlay combinatorics and keeps the phase-dependent
rendering co-located with the config injection logic.

### Deployment Strategy

Use Recreate only for transitions that change write acceptance:

| Phase | Strategy |
| --- | --- |
| PreparingKey | Rolling |
| EnteringReadOnly | Recreate |
| ReadOnly | Rolling |
| ExitingReadOnly | Recreate |

**Strategy is derived from the target phase (returned by
`selectReadOnlyPhase()`) on every reconcile, not carried forward from the
previous deployment state.** This ensures idempotent recovery if the operator
crashes or restarts mid-transition: the next reconcile re-derives the correct
strategy from the target phase without inspecting the live deployment's
strategy field.

When setting Recreate:

```go
deployment.Spec.Strategy = appsv1.DeploymentStrategy{
    Type: appsv1.RecreateDeploymentStrategyType,
}
```

`RollingUpdate` must be nil.

When restoring Rolling:

- `quay-app`: use Kubernetes default RollingUpdate behavior.
- `quay-mirror`: restore `maxUnavailable: 0`, `maxSurge: 1`.

During `EnteringReadOnly` and `ExitingReadOnly`, set
`terminationGracePeriodSeconds: 120` for `quay-app` and `quay-mirror`. Large
blob uploads (multi-GB layers) can take minutes; 120 seconds provides a
reasonable drain window. In-flight uploads exceeding this window will be
interrupted; document this. The 120-second grace period also applies during
abort rollouts from `EnteringReadOnly` (where the Recreate strategy is in
effect and pods may have been processing uploads). Since
`terminationGracePeriodSeconds` is derived from the target phase and
abort-from-EnteringReadOnly still uses Recreate, the grace period is
preserved through the abort.

### HPA Interaction

When HPA is managed (`ComponentHPA`), the HPA controls replica counts
independently. During `EnteringReadOnly` and `ExitingReadOnly` with Recreate
strategy, all pods are killed simultaneously.

If HPA is managed:

- **Before rendering pinned HPA bounds**, persist the original HPA bounds. Store
  the original `minReplicas` and `maxReplicas` in annotations on the HPA object:

  ```yaml
  annotations:
    quay-operator/readonly-original-min-replicas: "2"
    quay-operator/readonly-original-max-replicas: "20"
  ```

  The operator's default HPA bounds are `minReplicas: 2`, `maxReplicas: 20`
  (from `kustomize/components/horizontalpodautoscaler/`). The operator does
  not expose a supported CR API for customizing HPA bounds — it server-side-
  applies managed HPAs with force ownership, and replica overrides are
  rejected when HPA is managed. However, live HPA values may differ from
  the Kustomize defaults (e.g., manual kubectl edits, previous operator
  versions). Reading from the live HPA before pinning is defensive: it
  avoids damaging existing state, but does not declare manual HPA edits as
  a supported API.

  **HPA identification:** Identify HPAs by `scaleTargetRef` (targeting the
  `quay-app` or `quay-mirror` Deployment), not only by `quay-component`
  label. The `quay-app` HPA has `quay-component: horizontalpodautoscaler`
  while the mirror HPA has `quay-component: mirror`.

- During `EnteringReadOnly` and `ExitingReadOnly`, the **rendered desired HPA**
  must set `minReplicas = maxReplicas = pinned count` to prevent HPA scaling
  during the transition. Do not rely on a one-time imperative HPA patch: the
  controller server-side-applies the rendered HPA every reconcile, so pinning
  must be part of desired state or it can be overwritten by `Inflate()/Apply()`.
  The pinned count is determined by this fallback order:
  1. `HPA.status.desiredReplicas` — the HPA's current scaling decision,
     reflecting actual load. This is the primary source.
  2. If `HPA.status.desiredReplicas` is 0 or unavailable (HPA not yet
     initialized): use `Deployment.status.replicas` — the currently running
     count.
  3. If the Deployment has no running replicas: use `HPA.spec.minReplicas`
     (the configured floor, default 2).
  Test all three fallback scenarios: HPA scaled up (`desiredReplicas > min`),
  HPA at minimum, and HPA not yet initialized.
- Pinning is a separate reconcile step before the Recreate deployment rollout.
  **Concrete gate:** Within a single reconcile pass, the operator makes
  multiple sequential Apply calls (not one batched Apply for all objects):
  1. Save original bounds annotations if missing.
  2. Apply ONLY the pinned HPA (via direct SSA patch). Do NOT include the
     Recreate Deployment in this Apply.
  3. Re-read the live HPA from the API server and verify
     `minReplicas == maxReplicas == pinned count`.
  4. If verification fails (e.g., HPA controller has not reconciled yet),
     **requeue without applying the Deployment**. The next reconcile re-checks.
  5. Only after verification passes, apply the Recreate Deployment in the
     same or next reconcile.
  This ordering prevents the autoscaler from changing replicas while the
  operator is about to kill all pods with Recreate. The key implementation
  detail: `Inflate()` returns all objects, but the reconciler's Apply loop
  must defer the Deployment apply when HPA pinning verification has not
  passed. Use a flag or substatus marker (e.g., an annotation on the HPA:
  `quay-operator/readonly-hpa-pinned: "true"`) to track pinning state across
  reconcile boundaries.
- After the transition completes and strategy reverts to Rolling, render the HPA
  with `minReplicas` and `maxReplicas` restored from the saved annotations, then
  remove the annotations.
- If the annotations are missing on restore (e.g., operator crash before the
  annotation write completed), fall back to the Kustomize-rendered HPA values
  from `Inflate()`.
- If HPA has scaled `quay-app` to 0 (unlikely with typical `minReplicas >= 1`
  but possible with custom overrides), set `ReadOnlyDeferred` as with the
  zero-replicas case.

**Mirror HPA:** The operator also creates an HPA for `quay-mirror`
(`kustomize/components/horizontalpodautoscaler/mirror.horizontalpodautoscaler.yaml`,
default `minReplicas: 2`, `maxReplicas: 20`). Since `quay-mirror` also
transitions with Recreate strategy during `EnteringReadOnly` and
`ExitingReadOnly`, the mirror HPA must be pinned and restored using the same
annotation-based flow as `quay-app`:

- Same annotations: `quay-operator/readonly-original-min-replicas`,
  `quay-operator/readonly-original-max-replicas` on the mirror HPA object.
- Same pinned count fallback order (`HPA.status.desiredReplicas` →
  `Deployment.status.replicas` → `HPA.spec.minReplicas`).
- Same rendered desired-state pin/verify/restore logic after transition
  completes.
- Only apply if `ComponentIsManaged(..., ComponentMirror)` is true.

## Cleanup Job

### Purpose

The operator never directly mutates the Quay DB. It creates a short-lived
Quay-image Job that expires the operator-managed readonly key.

### When It Runs

- Normal exit from `ReadOnly`.
- Abort after `PreparingKey` created a Secret/key.
- Abort from `EnteringReadOnly` (see "Abort From EnteringReadOnly" below).
- Cleanup during `QuayRegistry` deletion while `readOnlyPhase != ""`.
- Recovery from missing Secret during exit, using `status.readOnlyKeyID`.

### Job Command

Use the actual image from the rendered `quay-app` Deployment.

**Entrypoint consideration:** The existing upgrade Job uses
`args: ["migrate", "head"]`, preserving the image entrypoint
(`quay-entrypoint.sh`). The `migrate` mode runs `certs_install.sh` and
`client_certs.sh` before executing the migration — these scripts set up DB
TLS certificates, proxy CA bundles, and client cert stores that are required
for database connectivity.

Using `command: ["python", ...]` would override the entrypoint and bypass
these init scripts, breaking the cleanup Job in environments that require DB
TLS or custom CA bundles.

**Recommended approach:** Add a new entrypoint mode for the cleanup CLI.
Add a `"servicekey-expire"` case to `quay-entrypoint.sh`:

```bash
"servicekey-expire")
    echo "Expiring service key"
    "${QUAYPATH}/conf/init/certs_install.sh" || exit
    "${QUAYPATH}/conf/init/client_certs.sh" || exit
    shift
    PYTHONPATH="${PYTHONPATH}:${QUAYPATH}" python \
        "${QUAYPATH}/tools/manage_servicekey.py" expire "$@"
    ;;
```

**Note on `shift`:** The existing entrypoint reads `QUAYENTRY=$1` but
never calls `shift`, so `"$@"` still contains the mode name as `$1`. Other
modes sidestep this by using positional args directly (`$2` in `migrate`), but
the `servicekey-expire` branch must pass remaining args to the CLI. Add an
explicit `shift` before `"$@"` so the CLI receives `--kid <kid> --grace-seconds
<grace>`, not `servicekey-expire --kid <kid> ...`.

Job spec uses `args` (not `command`) to preserve the entrypoint:

```yaml
args:
  - servicekey-expire
  - --kid
  - <status.readOnlyKeyID>
  - --grace-seconds
  - <computed grace>
backoffLimit: 3
restartPolicy: OnFailure
```

`backoffLimit: 3` limits internal Job retries. The operator handles
higher-level retries at the reconcile level (delete failed Job, recreate on
next reconcile). A low limit ensures the operator regains control quickly
rather than waiting for Kubernetes to exhaust retries with exponential backoff.

Required environment:

```yaml
env:
  - name: PYTHONPATH
    value: /quay-registry
  - name: QUAYPATH
    value: /quay-registry
  - name: QUAYCONF
    value: /quay-registry/conf
  - name: QE_K8S_CONFIG_SECRET
    value: <rendered config secret name>
  - name: QE_K8S_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: QUAY_OVERRIDE_CONFIG
    value: '{"REGISTRY_STATE":"normal"}'
```

The cleanup Job must mirror the existing upgrade Job pattern
(`kustomize/components/job/quay.upgrade.job.yaml`). Explicitly inherit:

- service account: same as `quay-app`
- volumes and volume mounts: `config` (rendered config Secret), `extra-ca-certs`,
  `postgres-certs`, `postgres-certs-store` — identical to the upgrade Job
- proxy env vars: `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` from the proxy
  config Secret
- security context: match quay-app container security context
- resource requests/limits: `requests: {cpu: 200m, memory: 256Mi}`,
  `limits: {cpu: 500m, memory: 512Mi}`. These accommodate Quay's Python
  process startup overhead (cert init scripts, config loading) while being
  appropriately sized for a single DB API call workload

The cleanup Job does not need the readonly private key Secret mounted.

**Env override isolation:** The operator middleware (`pkg/middleware/middleware.go`)
applies `spec.components[].overrides.env` for `ComponentQuay` to ALL Jobs
indiscriminately. If middleware applies a user-provided
`QUAY_OVERRIDE_CONFIG` after the cleanup Job is built, it can replace the
operator's required cleanup override (`REGISTRY_STATE: normal`) and break
cleanup.

The cleanup Job must be exempt from user env overrides. Implementation options:

- **Recommended:** Add the cleanup Job AFTER middleware processing. The
  operator constructs the cleanup Job programmatically in the reconcile loop
  (not via Kustomize), so it is not part of the `Inflate()` output that
  passes through `middleware.Process()`. This naturally exempts it.
- **Alternative:** Add a label or annotation (`quay-operator/skip-overrides:
  "true"`) and check it in middleware before applying env overrides.

The cleanup Job's `QUAY_OVERRIDE_CONFIG` env var must be the final value —
test that user `ComponentQuay` env overrides do not clobber it. If an inline
user `QUAY_OVERRIDE_CONFIG` was allowed because it does not set
`REGISTRY_STATE`, the cleanup Job must preserve those keys by merging them with
`REGISTRY_STATE: normal`.

**Config Secret lifetime:** The cleanup Job references the rendered config
Secret via `QE_K8S_CONFIG_SECRET`. The operator normally deletes old rendered
config Secrets after the `quay-app` Deployment has fully rolled out
(`quayAppDeploymentRolledOut()` gates cleanup, per PROJQUAY-9157). During
`ExitingReadOnly`, the rendered config changes (removing `REGISTRY_STATE`),
which produces a new hash-suffixed Secret name.

**Required ordering:** The cleanup Job must reference the NEW rendered config
Secret (with `REGISTRY_STATE` removed), not the old one. The cleanup Job uses
`QUAY_OVERRIDE_CONFIG={"REGISTRY_STATE":"normal"}` to override the registry
state independently, so it does not need the old config with `REGISTRY_STATE:
readonly`. The ordering is:

1. `Inflate()` renders the new config Secret (without `REGISTRY_STATE`,
   key path overrides, import flag, or readonly expiration override —
   matching the ExitingReadOnly config rendering rules above).
   The reconciler discovers the rendered config Secret name by scanning
   the inflated objects for the generated Quay config Secret, reusing the
   existing controller pattern. It must not reimplement Kustomize hash
   logic.
2. `Apply()` creates the new config Secret and rolls the Deployment.
3. Wait for Deployment rollout (subsequent reconciles).
4. Create the cleanup Job with `QE_K8S_CONFIG_SECRET` set to the new config
   Secret name. The Job only needs DB connectivity and general config from
   the new Secret; `QUAY_OVERRIDE_CONFIG` handles the readonly bypass.
5. Defer old config Secret deletion while `readOnlyPhase` is
   `ExitingReadOnly` and the cleanup Job is still active. The old-Secret
   cleanup code (`quayAppDeploymentRolledOut()` gate) must additionally
   check for active cleanup Jobs before deleting.

Test: verify that old rendered config Secrets are not deleted while a cleanup
Job is running.

### Grace Period

Compute grace from the effective config (source bundle overlaid with allowed
inline `QUAY_OVERRIDE_CONFIG` values, then explicit Quay defaults):

```python
grace = max(
    SIGNED_GRANT_EXPIRATION_SEC or 86400,
    REGISTRY_JWT_AUTH_MAX_FRESH_S or 3660,
    86400,
)
```

Use the computed value everywhere. Do not hardcode `86400` in Job command
examples except as the minimum floor.

**How the operator reads config values:** The reconciler computes grace from
the effective config: the source config bundle overlaid with any allowed
inline `QUAY_OVERRIDE_CONFIG` values, then explicit Quay defaults.

`QUAY_OVERRIDE_CONFIG` has higher precedence than rendered `config.yaml`
(`app.py`). If a user sets `SIGNED_GRANT_EXPIRATION_SEC` or
`REGISTRY_JWT_AUTH_MAX_FRESH_S` via an allowed inline override, Quay applies
those values at runtime. The grace computation must account for this or it
could expire the key too early — the cleanup grace window would be shorter
than the actual token lifetime Quay is enforcing.

A standalone helper computes the effective grace:

```go
func graceSeconds(bundle *corev1.Secret, quay *v1.QuayRegistry) int {
    cfg := map[string]interface{}{}
    if err := yaml.Unmarshal(bundle.Data["config.yaml"], &cfg); err != nil {
        return 86400
    }
    if cfg == nil {
        cfg = map[string]interface{}{}
    }

    // Overlay allowed inline QUAY_OVERRIDE_CONFIG values (if any).
    // The override conflict check has already validated that no
    // blocked readonly lifecycle keys are present in the override.
    if overrideCfg := getAllowedInlineOverrideConfig(quay); overrideCfg != nil {
        for k, v := range overrideCfg {
            cfg[k] = v
        }
    }

    return max(
        intOr(cfg, "SIGNED_GRANT_EXPIRATION_SEC", 86400),
        intOr(cfg, "REGISTRY_JWT_AUTH_MAX_FRESH_S", 3660),
        86400,
    )
}
```

The reconciler calls this only during `ExitingReadOnly` when constructing the
cleanup Job command. If a key is absent or invalid, that key uses its explicit
Quay default (`SIGNED_GRANT_EXPIRATION_SEC=86400`,
`REGISTRY_JWT_AUTH_MAX_FRESH_S=3660`). If the config cannot be parsed at all,
the helper uses the default/floor result of `86400` seconds.

This requires no changes to `Inflate()`, its return type, or
`QuayRegistryContext`.

### Failure Handling

If the Job fails:

- do not delete the readonly Secret
- do not clear `status.readOnlyKeyID`
- do not advance to Normal
- set `ReadOnly=False/ReadOnlyCleanupFailed` (the registry has rolled out to
  normal mode; only credential cleanup is blocked — see condition table)
- emit Warning event
- requeue with backoff
- on next reconcile, delete the failed Job and recreate it

If the registry has already rolled out to normal mode but cleanup fails, use
`ReadOnlyCleanupFailed`, not `ReadOnlyDegraded`, because the registry state is
normal but credential cleanup is blocked.

### Finalizer Cleanup

During `QuayRegistry` deletion, finalizer cleanup must run before removing the
operator finalizer if:

```text
status.readOnlyPhase != "" && status.readOnlyKeyID != ""
```

Because the CR is deleting, do not create a cleanup Job with an ownerReference
to the deleting CR. Create the Job with no ownerReference and track it by a
well-known label:

```yaml
labels:
  quay-operator/cleanup-for: <quayregistry-name>
  quay-operator/cleanup-kid: <status.readOnlyKeyID>
```

The operator finds existing cleanup Jobs by label selector before creating a
new one. After the Job succeeds, the operator deletes it explicitly, then
removes the finalizer.

**Config Secret for finalizer cleanup:** The deletion path
(`manageQuayDeletion()`) runs before the normal Inflate/Apply path, so no
new config Secret is rendered during deletion. The finalizer cleanup Job
must use the **current rendered config Secret** (which may still contain
`REGISTRY_STATE: readonly`) plus
`QUAY_OVERRIDE_CONFIG={"REGISTRY_STATE":"normal"}` to bypass readonly at
runtime. This is the same override mechanism used for normal-exit cleanup
and is already validated by the cleanup CLI design. Do not render a
finalizer-specific config Secret — the current Secret has all the DB
connectivity and TLS config the Job needs.

**Timeout:** Apply a hard timeout of 10 minutes on finalizer cleanup. Store
the cleanup start time as an annotation on the QuayRegistry CR
(`quay-operator/cleanup-started-at`) when the first cleanup Job is created.
This timestamp survives both operator restarts and Job deletion/recreation
(unlike Job `creationTimestamp`, which resets on recreation and could extend
deletion indefinitely). On each finalizer reconcile, compare `time.Now()`
against the annotation value.

**Recovery for missing or malformed annotation:** If a cleanup Job exists
but the annotation is missing (e.g., operator crashed between Job creation
and annotation write), backfill the annotation from the Job's
`metadata.creationTimestamp`. If the annotation value is unparseable, emit
a Warning event and treat the timeout as already exceeded — remove the
finalizer and let the orphaned key self-expire.

The current `manageQuayDeletion()` returns `(Result, error)` using
`r.Requeue` on error. The readonly finalizer cleanup path must return
`ctrl.Result{RequeueAfter: 30 * time.Second}` instead of `r.Requeue` while
waiting for the Job, so the reconciler polls at a known interval rather than
using the default error-requeue backoff.

If the cleanup Job does not complete within 10 minutes:

1. Log a Warning event explaining that cleanup timed out. Include the `kid`
   in the event message.
2. Remove the finalizer and allow CR deletion to proceed.
3. The orphaned key will self-expire. Because the design uses bounded
   expiration (`INSTANCE_SERVICE_KEY_EXPIRATION: 43200` minutes = 30 days),
   and the `ServiceKeyWorker` is no longer refreshing the key (the registry
   is being deleted), the key's `expiration_date` will pass within at most
   30 days of its last refresh. No manual intervention is required.
4. The orphaned cleanup Job will remain for manual inspection. Document that
   admins should check for Jobs with label `quay-operator/cleanup-for` if
   deletion seems to leave stale resources.

This avoids the CR being stuck in `Terminating` indefinitely if the cleanup Job
cannot reach the database or encounters persistent failures.

## Config Rendering

The operator modifies `parsedUserConfig` in `pkg/kustomize/kustomize.go` before
encoding `config.yaml`.

When `spec.readOnly` is nil and phase is normal, the operator preserves
manual readonly lifecycle fields in the source config except for
`INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES`. The import flag is operator-owned, so
source-provided values are always stripped unless the current operator phase
explicitly injects it. This prevents an administrator-provided config bundle
from enabling the normal-mode import branch without the readonly Secret mount.

When `spec.readOnly` is non-nil, the CR is authoritative:

- `false`: remove any user-provided `REGISTRY_STATE` and manual service-key
  path fields (`INSTANCE_SERVICE_KEY_KID_LOCATION`,
  `INSTANCE_SERVICE_KEY_LOCATION`) plus `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES`
- `true`: render per the phase matrix below

**Phase matrix for rendered config (when `spec.readOnly=true`):**

| Phase | `REGISTRY_STATE` | Key paths | Import flag | Expiration | Strip source `REGISTRY_STATE` |
| --- | --- | --- | --- | --- | --- |
| PreparingKey | omit | inject `/conf/readonly/` paths | `true` | `43200` | yes (defensive) |
| EnteringReadOnly | `readonly` | inject `/conf/readonly/` paths | `true` | `43200` | n/a (operator injects) |
| ReadOnly | `readonly` | inject `/conf/readonly/` paths | `true` | `43200` | n/a (operator injects) |
| ExitingReadOnly | remove | remove | remove | remove | n/a |
| Normal (`""`) | remove | remove | remove | remove | n/a |

Key paths = `INSTANCE_SERVICE_KEY_KID_LOCATION: /conf/readonly/quay-readonly.kid`
and `INSTANCE_SERVICE_KEY_LOCATION: /conf/readonly/quay-readonly.pem`.

The key paths and bounded expiration must be present in `EnteringReadOnly`
and `ReadOnly`, not only in `PreparingKey`. In readonly boot, Quay reads
these paths from config; if they are missing, it falls back to the defaults
under `CONF_DIR` (`/conf/stack/quay.kid`, `/conf/stack/quay.pem`) and will
fail boot or use the wrong key.

During `PreparingKey`, additionally strip any source-provided
`REGISTRY_STATE` from the rendered config. This is defensive — the manual
detection guard blocks entry when the source config contains
`REGISTRY_STATE: readonly`, but the rendering layer should be independently
safe as a defense-in-depth measure.

During `ExitingReadOnly`, do not delete the directly managed readonly Secret
until cleanup succeeds.

## Override Conflict Detection

`QUAY_OVERRIDE_CONFIG` has higher precedence than rendered `config.yaml`.

The operator must reject transitions if `QUAY_OVERRIDE_CONFIG` contains any
operator-owned readonly lifecycle key. Since Quay applies
`QUAY_OVERRIDE_CONFIG` after normal config loading (`app.py`), any of
these keys in the override would silently replace the operator-rendered
values at runtime.

Blocked keys during operator-managed readonly lifecycle:

- `REGISTRY_STATE`
- `INSTANCE_SERVICE_KEY_KID_LOCATION`
- `INSTANCE_SERVICE_KEY_LOCATION`
- `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES`
- `INSTANCE_SERVICE_KEY_EXPIRATION`

**v1 scope decision:** The operator currently has zero references to
`QUAY_OVERRIDE_CONFIG` — this is entirely new operator logic with no existing
pattern to follow. Building a full Secret/ConfigMap indexer for
`valueFrom`-based overrides is disproportionate for v1. Implement as a
standalone helper function for independent unit testability:

```go
func hasConflictingOverride(quay *v1.QuayRegistry, blockedKeys []string) (bool, string, error)
```

Returns `(blocked, reason, err)` where `reason` describes why the override
conflicts. The env override structure uses `corev1.EnvVar`, so parsing is
standard Kubernetes API. Pass the full blocked-keys list above.

For v1, check **only**:

- `spec.components[].overrides.env` entries where `name` equals
  `QUAY_OVERRIDE_CONFIG` and `value` is an inline string.
- Parse the inline JSON value. If it contains any blocked key, block with
  `OverrideConflict`.
- If the inline JSON value is valid and does not contain any blocked key, the
  transition may proceed. Preserve those allowed keys when constructing the
  cleanup Job by merging them with the operator-required
  `REGISTRY_STATE: normal` override. Do not preserve any of the blocked keys
  in the cleanup Job override.
- If `valueFrom` is used for `QUAY_OVERRIDE_CONFIG`, **block** the transition
  with `OverrideConflict`. The operator cannot inspect the referenced
  Secret/ConfigMap value, so it cannot guarantee that no blocked readonly
  lifecycle key is being overridden. A warning-only approach would risk
  producing a false `ReadOnlyActive` condition: the operator marks readonly
  as active while a `valueFrom` override silently changes an operator-owned
  key at runtime. Blocking is the safe default. The admin must remove the
  `valueFrom`-based `QUAY_OVERRIDE_CONFIG` env before enabling readonly.
  Document this requirement.
- Invalid JSON in an inline `value` is blocking.

Post-v1 enhancement: index referenced Secrets/ConfigMaps and resolve
`valueFrom` values. This requires a Secret/ConfigMap watcher, which is
tracked separately.

## Version and Image Compatibility

The operator must not enable mounted-key import for old Quay images that lack
the required `boot.py` changes (file-based key import, entrypoint
`servicekey-expire` mode, `manage_servicekey.py` CLI).

Rules:

1. `status.currentVersion == ""`: defer transition, keep normal reconcile
   running, set `ReadOnlyDeferred` or `UnsupportedVersion` with clear message.
2. Normalized `status.currentVersion < 3.19.0`: block with
   `UnsupportedVersion`.
3. Standard operator-managed image: allow if normalized
   `status.currentVersion >= 3.19.0`.
4. Custom image override: require explicit CR annotation whose value equals
   the full custom image reference (including tag or digest):

```yaml
metadata:
  annotations:
    quay.redhat.com/readOnlyCompatible: "registry.example.com/myquay:v3.19.0-custom"
```

The annotation is an admin assertion that **this specific image** contains
the required Quay changes. If the admin later changes the custom image
without updating the annotation, the operator treats the new image as
incompatible. A boolean `"true"` is not sufficient because it would carry
over to any subsequent custom image, defeating the non-normal-phase image
protection. The operator compares the annotation value against the
currently rendered `quay-app` container image; they must match exactly.
OCI label inspection may be added later but is not required for v1.

**Version parsing requirement:** `status.currentVersion` is stored as an opaque
string and may include a leading `v` or prerelease/build suffix
(for example, `v3.19.0`, `3.19.0`, or `v3.99.0-dev`). Do not compare it with
plain string ordering. Parse with the existing vendored
`github.com/Masterminds/semver/v3` package, using `semver.NewVersion()` and a
constraint equivalent to `>= 3.19.0-0` so leading `v` and prerelease/dev builds
are handled consistently. If parsing fails, block with `UnsupportedVersion`
and a message that includes the unparseable value.

**Image compatibility during non-normal phases:** The version/compatibility
gate applies to **all non-normal `readOnlyPhase` values**, not just entry.
The cleanup Job uses the rendered `quay-app` image and requires the
`servicekey-expire` entrypoint mode. If an admin changes to an incompatible
custom image while `readOnlyPhase` is non-normal:

- On each reconcile, re-check compatibility rules (version or annotation).
- If the image is no longer compatible, set `ReadOnly=True/ReadOnlyDegraded`
  (if in `ReadOnly`) or `ReadOnly=Unknown/UnsupportedVersion` (if in a
  transitional phase) with a message: "Current image does not support
  readonly lifecycle operations. Restore a compatible image or add the
  readOnlyCompatible annotation."
- Do not advance the phase. Do not create cleanup Jobs with an incompatible
  image.
- **Block the incompatible image rollout** for `quay-app` and `quay-mirror`
  Deployments: render the Deployments with the image stored in
  `status.readOnlyCompatibleImage` (recorded when transitioning from `""`
  to `PreparingKey`). This prevents the incompatible image from being
  deployed while the readonly lifecycle is active. The field persists
  across operator restarts because it is in the CR status.
- Once the admin restores a compatible image (or adds the
  `readOnlyCompatible` annotation), the operator resumes normal rendering
  with the new image and unfreezes the state machine.

**Recovery path:** The admin must either restore the previous compatible
image or set `quay.redhat.com/readOnlyCompatible` to the exact image ref. The
state machine then resumes from the current phase. No manual phase reset
is needed.

This ensures cleanup never fails silently due to a missing entrypoint mode,
and prevents an incompatible image from running while the readonly lifecycle
depends on features it lacks.

**Cleanup Job image during image freeze:** The image compatibility gate
checks the **rendered** Deployment image, not the admin's desired image. When
the Deployment is frozen to `status.readOnlyCompatibleImage`, the rendered
image IS compatible, so cleanup Job creation and recreation (after failure)
proceed normally using the frozen compatible image. The `UnsupportedVersion`
condition reflects the admin's desired image change but does not block
cleanup operations that use the frozen image. This prevents cleanup from
being stuck when a failed Job needs recreation while an incompatible image
change is pending.

## Edge Cases

### Manual Readonly Config

When `spec.readOnly=true` and the source config contains manual readonly
lifecycle fields (`REGISTRY_STATE: readonly`, `INSTANCE_SERVICE_KEY_*_LOCATION`,
`INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES`), the operator blocks with
`ManualMigrationRequired`. The admin must remove those fields from the source
config before enabling operator-managed readonly.

If `spec.readOnly` is nil, preserve manual readonly config unchanged.

If `spec.readOnly=false`, rendered config is normal/read-write regardless of
manual readonly lifecycle fields in the source config bundle.

### Abort From PreparingKey

Abort ordering must avoid pods importing the key after cleanup:

1. Render normal config without key paths/mount.
2. Roll pods off the readonly key mount.
3. Wait for rollout complete.
4. Run cleanup Job using `status.readOnlyKeyID`.
5. Delete Secret.
6. Clear `status.readOnlyKeyID`.
7. Reset phase.

**State-machine mechanics:** Abort from `PreparingKey` uses **inline
reconciler logic**, not a dedicated phase or `ExitingReadOnly`. The registry
was never in readonly mode during `PreparingKey` — `REGISTRY_STATE: readonly`
has not been injected yet — so `ExitingReadOnly` semantics (which assume
the registry IS readonly and needs a Recreate rollout back to normal) do not
apply. Instead, `selectReadOnlyPhase()` detects `spec.readOnly=false` +
`phase=PreparingKey` and executes the abort as a multi-reconcile sequence:

- Reconcile 1: Render normal config (remove key paths/mount from Inflate
  output). The strategy remains Rolling (PreparingKey uses Rolling). Requeue.
- Reconcile 2+: Check rollout complete: the live `quay-app` Deployment's pod
  template must NOT contain the `readonly-service-key` volume, AND
  `ObservedGeneration >= Generation` AND `UpdatedReplicas == Replicas` AND
  `AvailableReplicas == Replicas` (matching the existing operator rollout
  safety check pattern). Once
  these hold, look for an existing cleanup Job by label
  `quay-operator/cleanup-kid: <status.readOnlyKeyID>`. If none exists, create
  one. Requeue with `RequeueAfter: 30s`.
- Subsequent reconciles: Poll cleanup Job by label. If Job status is
  `Succeeded`: delete the readonly Secret (by name
  `<name>-readonly-service-key`), clear `status.readOnlyKeyID`, reset phase
  to `""`. If Job status is `Failed` with `backoffLimit` exhausted: delete
  the failed Job, set `ReadOnly=False/ReadOnlyCleanupFailed`, requeue. The
  next reconcile recreates the Job.

The abort MUST NOT skip steps 4-6 by resetting the phase directly. By the
time `PreparingKey` is active, the Secret exists and `boot.py` may have
already imported the key into the DB. Skipping cleanup leaves an orphaned
DB key row (which self-expires after 30 days via bounded expiration) and an
orphaned Kubernetes Secret (which is garbage-collected via ownerReference
when the CR is deleted, but persists indefinitely otherwise).

If the cleanup Job fails, follow the same failure handling as normal exit:
set `ReadOnly=False/ReadOnlyCleanupFailed`, requeue with backoff, delete
and recreate the Job on the next reconcile. The registry is already in
normal mode (writes are accepted) — only credential cleanup is blocked.

### Abort From EnteringReadOnly

If `spec.readOnly` changes to `false` or `nil` while `readOnlyPhase` is
`EnteringReadOnly`, the operator must abort the transition. The abort follows
the same pattern as abort from `PreparingKey`:

1. Render normal config without `REGISTRY_STATE: readonly`, without key
   paths/mount.
2. Roll pods to the new config (Recreate strategy is already set; the next
   rollout uses the normal config).
3. Wait for rollout complete.
4. Run cleanup Job using `status.readOnlyKeyID`.
5. Delete Secret.
6. Clear `status.readOnlyKeyID`.
7. Reset phase to `""`.

The key difference from `PreparingKey` abort is that some pods may have already
been running in readonly mode. The rollout back to normal config handles this.

### Full State Transition Table

Every combination of current phase and `spec.readOnly` value must have a
defined behavior. This table is authoritative:

| Current Phase | spec.readOnly=true | spec.readOnly=false | spec.readOnly=nil |
| --- | --- | --- | --- |
| `""` (Normal) | Begin → `PreparingKey` | Strip readonly lifecycle fields from rendered config; ensure normal mode | No-op (unmanaged) |
| `PreparingKey` | Continue preparing | Abort: roll pods off key, cleanup Job, delete Secret, reset to `""` | Abort: same as `false` |
| `EnteringReadOnly` | Continue entering | Abort: render normal config, wait rollout, cleanup Job, delete Secret, reset to `""` | Abort: same as `false` |
| `ReadOnly` | No-op (stable) | Begin exit → `ExitingReadOnly` | same as `false` (see note below) |
| `ExitingReadOnly` | Continue exit to `""`; re-enter on next reconcile if still `true` | Continue exiting | same as `false` |
| `ExitingReadOnly` + `CleanupFailed` | Continue exit to `""`; re-enter on next reconcile if still `true` | Retry cleanup Job, wait for success, then advance to `""` | same as `false` |

`CleanupFailed` only occurs during `ExitingReadOnly` — the registry has
already rolled out to normal mode but credential cleanup is blocked. It
cannot occur during stable `ReadOnly` because cleanup only runs on exit.
The condition is `ReadOnly=False/ReadOnlyCleanupFailed` (the registry is
NOT in readonly mode; see condition table).

**`spec.readOnly=nil` from a non-normal phase:** Once the operator enters
a non-normal `readOnlyPhase`, setting `spec.readOnly=nil` behaves the same
as `spec.readOnly=false` — the operator exits readonly, runs cleanup, and
resets the phase to `""`.

**Re-entering from `ExitingReadOnly`:** If `spec.readOnly` becomes `true`
while `readOnlyPhase` is `ExitingReadOnly` (or `ExitingReadOnly +
CleanupFailed`), continue the exit/cleanup to normal. After the phase resets
to `""`, the next reconcile sees `spec.readOnly=true` + phase `""` and begins
a fresh `PreparingKey` transition. This avoids the complexity of cancelling
an in-flight cleanup Job.

**Rapid toggle protection:** If `spec.readOnly` changes more than once within
a single reconcile interval (e.g., `true` → `false` → `true`), the operator
sees only the final value. The state machine advances from the current phase
toward the desired state on each reconcile. No special debounce is needed
because the phase is always re-evaluated from the current phase + current spec
on every reconcile.

### Operator Upgrade During ReadOnly

Operator upgrades while `readOnlyPhase` is non-empty are supported. The new
operator resumes from the persisted `status.readOnlyPhase`. All phase logic
must be idempotent: `selectReadOnlyPhase()` re-derives the target phase from
the persisted phase and `spec.readOnly` on every reconcile, and `Inflate()`
derives config, strategy, and mounts from that target phase — never from
cached state.

If the new operator version changes the `ReadOnly` condition reasons or phase
constants, ensure backward compatibility with phases written by the previous
operator version.

**Upgrade overlay and migration suppression:** While `readOnlyPhase != ""`,
the operator must suppress the Quay upgrade overlay and DB migration Job.
The existing `Inflate()` logic (`kustomize.go`) selects the `upgrade`
overlay when `status.currentVersion != QuayVersionCurrent` or when DB
config has changed. That overlay scales `quay-app` to 0 replicas and
creates a `quay-app-upgrade` migration Job — both of which conflict with
the readonly contract:

- Scaling to 0 makes the registry unavailable while backup tools expect
  `ReadOnlyActive` to mean the registry is serving reads.
- DB migrations write to the database while readonly is supposed to provide
  an application-consistent backup state.

The design **defers** rather than **blocks** the upgrade. The operator
binary has already been upgraded by OLM — that cannot be reversed. Blocking
(setting an error condition and requiring admin intervention) would force
manual steps after every backup window. Deferring resolves automatically:
after readonly exits, the next reconcile sees the version mismatch and runs
the pending migration with no admin action.

When `readOnlyPhase != ""`:

1. Skip the pre-render Postgres/Clair Postgres upgrade detection — do not
   call `checkNeedsPostgresUpgradeForComponent()`, do not set
   `NeedsPgUpgrade` / `NeedsClairPgUpgrade`, and do not scale Postgres
   deployments to 0. This function runs before `Inflate()` and would
   actively scale down DB components, conflicting with `ReadOnlyActive`
   backup expectations.
2. Skip the upgrade overlay selection — use the normal overlay even if
   `status.currentVersion != QuayVersionCurrent` or DB config has changed.
3. Do not create or run the `quay-app-upgrade` migration Job.
4. Bypass **both** the top-of-reconcile guards and the post-apply
   migration-wait return paths. The top-of-reconcile guards
   (`PostgresUpgradeRunning()`, `MigrationsRunning()`) check
   `ComponentsCreated` condition reasons (`MigrationsInProgress`,
   `PostgresUpgradeInProgress`, `PostgresUpgradeFailed`) and return early
   before config bundle loading, phase selection, or readonly health checks
   can execute. If a stale migration condition exists on the CR status (e.g.,
   set by a previous operator version or before readonly was entered), it
   would short-circuit every reconcile indefinitely. When
   `readOnlyPhase != ""` and no actual migration/upgrade Job is active,
   skip these guards and continue with readonly reconciliation.
5. Keep the Deployment image frozen to `status.readOnlyCompatibleImage`
   (already specified by Version and Image Compatibility).
6. After the readonly lifecycle exits to normal (`readOnlyPhase == ""`),
   re-enable all paths: Postgres upgrade detection, upgrade overlay,
   migration Job creation, and migration-wait returns. The next reconcile
   re-evaluates version/DB-config and runs any pending migrations.

### Zero Replicas

If desired `quay-app` replicas are zero, do not advance. Set
`ReadOnlyDeferred`. The operator cannot prove key import or mode transition with
zero app pods.

## Status Controller Interaction

The operator runs a separate `QuayRegistryStatusController` that reconciles
independently every minute via `cmpstatus.Evaluate()`. This controller
overwrites conditions for all known component types.

All readonly-related status fields must be **preserved** by the status
controller. The status controller's `Evaluate()` function must pass through
the following unchanged — they should not be included in the component
evaluation pipeline or overwritten during status reconciliation:

1. `ReadOnly` condition (type, status, reason, message, timestamps)
2. `status.readOnlyPhase`
3. `status.readOnlyKeyID`
4. `status.readOnlySuppressManualConfig`
5. `status.readOnlyCompatibleImage`

These fields are owned exclusively by the main reconciler. The status
controller must read-and-preserve them when writing its own condition
updates.

Register the `ReadOnly` condition type in:

1. `RemoveUnusedConditions()` — so it is not pruned.
2. The status controller's condition management — as a pass-through type that
   is never overwritten.

**Preservation mechanism:** The status controller's `overwriteConditions()`
uses `SetCondition()` to upsert each computed component condition into the
existing conditions slice, then calls `RemoveUnusedConditions()` to prune
unknown types. The `ReadOnly` condition is preserved by:

1. Adding `ReadOnly` to the `RemoveUnusedConditions()` allowlist so it is not
   pruned.
2. Ensuring `overwriteConditions()` never computes or overwrites a `ReadOnly`
   condition — it only processes component-health conditions. Since
   `SetCondition()` upserts by type, and `overwriteConditions()` never calls
   it with a `ReadOnly` type, the existing `ReadOnly` condition in the slice
   remains untouched.

Do NOT unconditionally append the `ReadOnly` condition in
`overwriteConditions()` — `SetCondition()` upserts by type, so a duplicate
append would create two `ReadOnly` entries in the conditions slice.

Test: exercise concurrent main reconciler + status controller updates and
verify all five readonly status fields are never transiently lost or
overwritten. Specifically: set `ReadOnly=True/ReadOnlyActive`, trigger a
status controller reconciliation, verify the `ReadOnly` condition persists
with unchanged values, timestamps, and message.

## Backup Integration

The primary use case for this feature is application-consistent backups.
Backup tools (Velero, OADP, custom controllers) can use the following
pattern:

1. Set `spec.readOnly=true` on the `QuayRegistry` CR.
2. Wait for `status.readOnlyPhase == "ReadOnly"` AND `ReadOnly` condition
   `status=True`, `reason=ReadOnlyActive`.
3. Perform backup.
4. Set `spec.readOnly=false`.

`ReadOnlyDegraded` (also `status=True`) is NOT a safe backup signal — backup
tools should require `reason=ReadOnlyActive` specifically.

## Observability

Add Prometheus metrics for monitoring readonly state across a fleet:

Required for v1:

- `quay_operator_readonly_active` (gauge, 0/1): `1` when ALL of:
  - `readOnlyPhase == "ReadOnly"`
  - `ReadOnly` condition `status == True`
  - `ReadOnly` condition `reason == ReadOnlyActive`

  `0` in all other states, including `ReadOnlyDegraded` (which has
  `status=True` but `reason != ReadOnlyActive`). This three-way check
  matches the backup integration guidance and ensures backup controllers
  relying on the metric do not proceed during degraded readonly. Labels:
  `quayregistry` (CR name), `namespace`.

  **Implementation notes:** The operator currently has no custom Prometheus
  gauges. Register using `controller-runtime`'s `metrics.Registry`. Update
  the gauge inside `advanceReadOnlyPhase()` on every reconcile. On CR
  deletion (finalizer), delete the label values for the deleted CR to clear
  stale series.

Post-v1:

- `quay_operator_readonly_transition_duration_seconds` (histogram): time spent
  in each transition phase.
- `quay_operator_readonly_cleanup_failures_total` (counter): number of cleanup
  Job failures.

The gauge metric is critical for the backup integration use case — backup
controllers can scrape it to determine when readonly is stable without
requiring Kubernetes API watch permissions.

## Security Notes

- Private key material exists only in the Kubernetes Secret and mounted files.
- The database must store public JWK only. Authlib's `JsonWebKey.as_dict()`
  returns only public RSA fields (`kty`, `n`, `e`, `kid`) — private fields
  (`d`, `p`, `q`, `dp`, `dq`, `qi`) are excluded by default.
  **Pre-existing gap:** The keyserver PUT endpoint (`_validate_jwk()`) only
  checks for required public fields but does not strip or reject private
  fields. It accepts both RSA and EC key types. An external caller could
  submit a JWK containing private material, and it would be stored and
  returned by the unauthenticated GET endpoint.
  **Required fix:** Add a shared public-JWK-only sanitizer that strips
  private fields for all supported key types before storage:
  - RSA: strip `d`, `p`, `q`, `dp`, `dq`, `qi`
  - EC: strip `d`

  Apply the sanitizer at the model/storage boundary:
  `create_service_key()` and `replace_service_key()` in
  `data/model/service_keys.py` must strip private fields from the `jwk`
  dict before writing to the database. This covers all storage paths
  (keyserver PUT, `generate_service_key()`, and the new import path)
  regardless of which caller provides the JWK. `_validate_jwk()` in the
  keyserver endpoint is only called by PUT and does not cover internal
  callers like `generate_service_key()` or the import path.
  Include a regression test asserting no private fields are present in
  the stored `ServiceKey.jwk` column or in the keyserver GET response,
  for both RSA and EC key types.
- The keyserver can safely return `ServiceKey.jwk` because private JWK fields
  are stripped on storage.
- The readonly Secret should be immutable and RBAC-protected.
- Cleanup expires the DB row after token drain; it does not immediately delete
  the DB row.
- Kubernetes stale Secret cleanup removes private material after cleanup Job
  success.

## Testing Requirements

### `quay/quay`

- `boot.py`: numeric expiration preserves current behavior.
- `boot.py`: mounted key files missing falls through to current TTL behavior
  only when `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` is false. When the import
  flag is true, missing files fail boot.
- `boot.py`: bounded expiration with mounted key files imports key from files
  when `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` is true.
- `boot.py`: when `REGISTRY_STATE == "readonly"`, startup skips
  `sync_database_with_config()` and `set_region_release()` so boot does not
  attempt DB writes. Include a missing-storage-location case that would have
  called `ImageStorageLocation.insert_many(...)` in normal mode.
- Config schema/defaults: `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` exists in
  `DefaultConfig`, is valid boolean config schema, and is listed in
  `INTERNAL_ONLY_PROPERTIES`.
- Import rejects `.kid` not matching computed thumbprint.
- Import rejects existing DB row with mismatched `n/e`.
- Import accepts existing row with valid bounded expiration and matching key.
- Import refreshes an existing expired row when the row is operator-managed and
  its public JWK matches the mounted PEM-derived JWK. Readonly verification
  still rejects expired rows because it cannot write the DB.
- Import performs direct `kid`-only lookup (not `kid + service`, since
  `ServiceKey.kid` is globally unique) before any path that triggers
  `_gc_expired(service)`, then validates `service` explicitly. This ensures a
  stale expired matching row is refreshed or rejected by ownership/JWK/service
  checks instead of being physically deleted by GC first, and produces a clean
  "wrong service" error instead of routing through `IntegrityError` recovery.
- With `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` unset/false, existing default
  `/conf/stack/quay.kid` and `/conf/stack/quay.pem` files do not trigger import
  or `created_by` backfill; normal boot keeps the current generate/write
  behavior.
- With `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES=true`, missing or invalid key
  files fail boot with a clear error and do not fall back to generating a
  replacement key.
- Import rejects existing row with wrong service.
- Import backfills `created_by` metadata on existing row that lacks it.
- Import handles concurrent `IntegrityError` and validates winner row.
- Import handles concurrent `ServiceKeyAlreadyApproved` during approval:
  catches the exception, re-reads the key, verifies approval exists, and
  treats it as success. Test two import callers racing on an unapproved
  existing key and on a freshly created key.
- Readonly verification rejects wrong service, unapproved, expired, missing PEM,
  mismatched PEM, and mismatched computed kid.
- Service key worker refreshes normally with bounded expiration (43200 min).
- Keyserver `GET /services/<service>/keys/<kid>` still returns raw JWK (no
  envelope), with `service` filtering fix applied.
- Keyserver `GET /services/<service>/keys/<kid>/status` returns `kid`,
  `service`, `operator_managed` (boolean), `expiration_date` (RFC 3339 UTC
  with `Z` suffix).
- `/status` does not expose full `metadata` — only the derived
  `operator_managed` boolean.
- `/status` returns `expiration_date: null` for keys with no expiration.
- `/status` returns 200 with full status payload for **expired** keys (unlike
  raw GET which returns 403). Verify the response includes the past
  `expiration_date`.
- `/status` returns 200 for keys expired longer than
  `EXPIRED_SERVICE_KEY_TTL_SEC` (7 days), as long as the DB row still exists.
  This verifies the dedicated lookup bypasses stale-expired filtering.
- `/status` returns 200 with full status payload for **unapproved** keys
  (unlike raw GET which returns 409).
- `/status` 404s for unknown kid, filters by service.
- Operator never treats `/status` 200 alone as proof that a key is
  approved/alive — approval and liveness are checked via the raw GET
  endpoint in step 1 of the two-step flow.
- `manage_servicekey expire`:
  - missing key is no-op
  - wrong service fails
  - existing key expiration is updated
  - already expiring is idempotent
  - works with readonly rendered config when cleanup override/bypass is present
  - refuses to expire key without `metadata.created_by == "quay-operator-readonly"`
  - accepts key with correct `created_by` metadata marker
- Import stores `created_by: quay-operator-readonly` in `ServiceKey.metadata`.
- Import auto-approves the key via `ServiceKeyApprovalType.AUTOMATIC`.
- **Private JWK stripping:** Verify that `create_service_key()` and
  `replace_service_key()` strip private fields for all supported key types
  before storage: RSA (`d`, `p`, `q`, `dp`, `dq`, `qi`) and EC (`d`). Cover
  all creation paths: keyserver PUT (self-signed and rotation),
  `generate_service_key()`, and the new import path. Assert the stored
  `ServiceKey.jwk` column contains only public fields. Also verify the raw
  GET keyserver response does not contain private fields when a JWK with
  private material was submitted via PUT. Test both RSA and EC key types.

### `quay-operator`

- CRD includes `spec.readOnly`, `status.readOnlyPhase`,
  `status.readOnlyKeyID`, `status.readOnlySuppressManualConfig`,
  `status.readOnlyCompatibleImage`, condition type, and reason constants.
- `readOnlySuppressManualConfig` is always `false` (reserved for future use).
  Status controller preserves all readonly status fields.
- `ReadOnlyActive` emits Normal event even with status True.
- `ReadOnlyCleanupFailed` emits Warning event.
- `ensureReadOnlyKeySecret()` creates immutable Secret and sets
  `status.readOnlyKeyID`.
- Existing Secret reuse sets missing `status.readOnlyKeyID`.
- Secret data drift or replacement during `PreparingKey` sets
  `ReadOnly=Unknown/ReadOnlyTransitioning` and does not advance.
- Config rendering injects/removes `REGISTRY_STATE`, key paths, bounded
  expiration, and `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` per phase.
- Deployment strategy switches to Recreate with nil `RollingUpdate`.
- Rolling strategy restores app and mirror values correctly.
- Phase advancement:
  - `PreparingKey` requires at least one available app pod AND two-step
    keyserver verification: raw GET returns 200 (approved/alive), `/status`
    returns `operator_managed=true`, PEM-derived `kid`/`n`/`e` match, and
    `expiration_date` is parseable as RFC 3339.
  - PreparingKey validation failures set `ReadOnly=Unknown` (not
    `True/ReadOnlyDegraded`) because the registry is not yet readonly.
  - Enter/exit require all desired `quay-app` (and managed `quay-mirror`)
    pods available
  - zero replicas defers
- Cleanup Job uses actual app image and required env/mounts matching the
  upgrade Job pattern.
- Cleanup Job failure does not delete Secret or clear key id.
- Missing Secret exit uses `status.readOnlyKeyID`.
- Abort from `PreparingKey` rolls pods off mount before cleanup.
- Abort from `EnteringReadOnly` renders normal config and runs cleanup.
- Finalizer cleanup runs and waits for Job completion during CR deletion.
- Finalizer cleanup times out after 10 minutes and allows CR deletion.
- `QUAY_OVERRIDE_CONFIG` detection covers inline `value` in
  `spec.components[].overrides.env`. Blocks `valueFrom`-based
  `QUAY_OVERRIDE_CONFIG` with `OverrideConflict`.
- Version gate normalizes `status.currentVersion` with semver parsing, blocks
  old/unparseable versions, accepts leading `v` and prerelease/dev suffixes,
  and blocks custom images without compatibility annotation.
- Manual readonly lifecycle fields in source config are blocked with
  `ManualMigrationRequired` when `spec.readOnly=true`.
- Cleanup Job env is not overwritten by user `ComponentQuay` env overrides.
- Cleanup Job preserves allowed inline `QUAY_OVERRIDE_CONFIG` keys by merging
  them with `REGISTRY_STATE: normal`; all blocked readonly lifecycle keys
  (see Override Conflict Detection) in user overrides and `valueFrom`
  overrides remain blocked.
- HPA original bounds are saved to annotations before rendering pinned bounds and restored
  after transition.
- HPA pinning is rendered desired state during `EnteringReadOnly` and
  `ExitingReadOnly`, not a one-time post-apply patch. Verify server-side apply
  does not overwrite pinned `minReplicas=maxReplicas`.
- Recreate deployment rollout starts only after the live HPA has been re-read and
  verified pinned.
- Managed-HPA phase advancement uses the pinned count as desired replicas.
- State transition table: all phase×spec combinations produce correct behavior
  including rapid toggle and nil-from-non-normal-phase.
- Status controller preserves all five readonly status fields during
  independent status reconciliation: `ReadOnly` condition, `readOnlyPhase`,
  `readOnlyKeyID`, `readOnlySuppressManualConfig`, and
  `readOnlyCompatibleImage`.
- HPA is pinned during Recreate transitions and restored after.
- Deployment strategy is idempotently derived from phase on every reconcile.
- `quay_operator_readonly_active` Prometheus metric is emitted.

### Cross-Language Conformance

The operator generates the RSA key pair and computes the RFC 7638 JWK
thumbprint in Go. Quay validates and imports the key in Python using Authlib.
A mismatch in thumbprint computation between the two implementations would
cause boot.py to reject the operator-generated key.

**PEM format:** The operator MUST generate the PEM in **PKCS#1 format**
(`BEGIN RSA PRIVATE KEY`) using Go's `x509.MarshalPKCS1PrivateKey()`. Quay's
existing key serialization uses `serialization.PrivateFormat.TraditionalOpenSSL`
(PKCS#1) in `boot.py`. Go's default `x509.MarshalPKCS8PrivateKey()` produces
PKCS#8 (`BEGIN PRIVATE KEY`) — while Python's `load_pem_private_key()` can
load both formats, using PKCS#1 ensures consistency and avoids any risk of
format-dependent behavior differences in JWK derivation. The cross-language
test fixtures below MUST include both format variants to verify that JWK
thumbprints are identical regardless of PEM encoding.

- Add a shared test fixture: a known RSA PEM key with its pre-computed RFC
  7638 thumbprint (using SHA-256 over the canonical `{"e":...,"kty":"RSA",
  "n":...}` JSON).
- Go test: generate the thumbprint from the PEM and assert it equals the
  fixture value. Verify that `x509.MarshalPKCS1PrivateKey()` is used (not
  PKCS#8).
- Python test: compute the thumbprint via Authlib's `JsonWebKey` and assert
  it equals the same fixture value.
- Include at least two test vectors: one 2048-bit and one 4096-bit RSA key.
- Include a test that loads the same key in both PKCS#1 and PKCS#8 PEM
  encodings and verifies the derived JWK thumbprint is identical.

### Integration/E2E

- Full enter readonly through `spec.readOnly=true`; push fails, pull succeeds.
- Pod restart during readonly succeeds using durable Secret.
- Readonly survives longer than 120 minutes.
- Exit readonly through `spec.readOnly=false`; push succeeds again.
- Cleanup Job expires key and Secret is deleted.
- Existing `test_readonly_push_pull` remains passing.
- Abort from `EnteringReadOnly` by setting `spec.readOnly=false` mid-transition.
- Manual readonly lifecycle fields in source config blocked with
  `ManualMigrationRequired` when `spec.readOnly=true`.
- Keyserver backward compatibility: existing `GET /keys/services/quay/keys/<kid>`
  still returns raw JWK at top level (not envelope). New `/status` endpoint
  returns `kid`, `service`, `operator_managed`, `expiration_date`.
- Incompatible image change during ReadOnly: change to a custom image
  without the `readOnlyCompatible` annotation while in `ReadOnly`. Verify
  the operator blocks the image rollout (keeps the compatible image running),
  sets `ReadOnlyDegraded`, and freezes the state machine. Restore the
  compatible image or add the annotation, and verify the state machine
  unfreezes and exit proceeds normally.
- Old rendered config Secrets are not deleted while a cleanup Job is active.
- Pre-expiry warning fires when service key is within 7 days of expiration.
- Stable ReadOnly health check detects expired key (raw GET 403) and sets
  `ReadOnlyDegraded`.
- Stable ReadOnly health check detects unapproved key (raw GET 409) and
  sets `ReadOnlyDegraded`.
- Stable ReadOnly health check detects missing key (raw GET 404) and sets
  `ReadOnlyDegraded`.
- Missing Secret during ReadOnly sets `ReadOnlyDegraded` without generating
  a new key or changing `status.readOnlyKeyID`.
- Operator restart/crash mid-transition recovers to the correct phase.
- Operator upgrade while `ReadOnly`: no `quay-app` scale-to-zero, no
  `quay-app-upgrade` Job created, `ReadOnlyActive` remains valid.
- After `spec.readOnly=false` and cleanup completes, pending upgrade/migration
  proceeds normally.
- DB config change during readonly is deferred — no upgrade overlay, no
  migration Job until readonly exits.
- Stale `MigrationsInProgress` or `PostgresUpgradeInProgress` condition on
  CR status does not short-circuit readonly reconciliation when no actual
  migration/upgrade Job is active.
- Concurrent status controller preserves all five readonly status fields:
  `ReadOnly` condition, `readOnlyPhase`, `readOnlyKeyID`,
  `readOnlySuppressManualConfig`, and `readOnlyCompatibleImage`.
- Finalizer cleanup during CR deletion completes within timeout.
- Finalizer cleanup succeeds when the only rendered config Secret still
  contains `REGISTRY_STATE: readonly` — the Job uses
  `QUAY_OVERRIDE_CONFIG` to bypass.
- Backup integration: verify `ReadOnly` condition and `readOnlyPhase` are
  stable and queryable by external controllers.

## Design Decisions

- **Service key import guard:** Normal-mode file import is gated by
  `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES: true`. The flag defaults to `false`
  and is rendered only by the operator with the readonly Secret key paths. Quay
  must preserve existing normal restart behavior when the default
  `/conf/stack/quay.kid` and `/conf/stack/quay.pem` files exist.
- **Service key expiration model:** Bounded expiration (43200 minutes = 30
  days). NOT null. `INSTANCE_SERVICE_KEY_EXPIRATION` is in **minutes**
  The `ServiceKeyWorker` refreshes normally.
  Orphaned keys self-expire within 30 days of last refresh. No null-expiration
  code paths, no DB schema changes, no worker skip logic.
- **Cleanup Job readonly bypass:** Use `QUAY_OVERRIDE_CONFIG` env override.
  The `QUAY_OVERRIDE_CONFIG={"REGISTRY_STATE":"normal"}` approach works because
  the override is processed during app initialization before `configure()` sets
  the database readonly flag. No scoped model bypass is needed. The cleanup
  Job must be constructed after middleware processing to avoid user env
  overrides clobbering the cleanup env var (middleware applies `ComponentQuay`
  overrides to all Jobs indiscriminately). If the user has an allowed inline
  `QUAY_OVERRIDE_CONFIG` without any blocked readonly lifecycle key (see
  Override Conflict Detection), merge those keys into the cleanup override
  and let `REGISTRY_STATE: normal` win.
- **Cleanup Job entrypoint:** Use `args: ["servicekey-expire", "--kid", ...]`
  (not `command:`). Add a `servicekey-expire` mode to `quay-entrypoint.sh`
  that runs `certs_install.sh` and `client_certs.sh` before the CLI, matching
  the `migrate` mode pattern. Include `shift` before `"$@"` to strip the
  mode name — the existing entrypoint does not shift after reading `$1`.
- **Cleanup Job ownership for finalizer path:** No ownerReference. Track by
  label `quay-operator/cleanup-for: <cr-name>`. Delete manually after success.
  Hard timeout of 10 minutes. Orphaned keys self-expire (bounded expiration).
- **v1 override conflict detection:** Check inline `value` in
  `spec.components[].overrides.env`. Block `valueFrom`-based
  `QUAY_OVERRIDE_CONFIG` with `OverrideConflict` (cannot inspect, risk of
  false `ReadOnlyActive`). No Secret/ConfigMap indexer in v1.
- **v1 custom image compatibility rule:** Explicit CR annotation
  `quay.redhat.com/readOnlyCompatible: <exact-image-ref>`. Document as a
  supported API. Value must match the rendered quay-app image exactly.
  Version gate is about mounted-key import support, not null expiration. Parse
  `status.currentVersion` with `github.com/Masterminds/semver/v3` using
  `semver.NewVersion()` and a constraint equivalent to `>= 3.19.0-0`; block
  unparseable values with `UnsupportedVersion`.
- **`manage_servicekey delete`:** Not included in v1. Operator only requires
  `expire`. `delete` can be added later for admin/test use with an explicit
  confirmation flag.
- **Manual readonly detection:** When `spec.readOnly=true` and source config
  contains manual readonly lifecycle fields, block with
  `ManualMigrationRequired`. Admin must remove those fields first.
- **Phase advancement verification:** Two-step check in `PreparingKey`:
  1. `GET /keys/services/quay/keys/<kid>` — existence, approval, liveness
     (fix `service` filtering). Response remains raw JWK (no breaking change).
  2. `GET /keys/services/quay/keys/<kid>/status` — new endpoint returning
     `kid`, `service`, `operator_managed`, `expiration_date`. Verify
     `operator_managed == true` and PEM/JWK consistency before advancing.
- **Operator-managed key identification:** Store
  `metadata.created_by = "quay-operator-readonly"` on the `ServiceKey` DB row
  via the `metadata` JSONField. Import backfills the marker on existing rows
  that match but lack it. CLI uses this to refuse expiring non-operator keys.
- **Cleanup Job config Secret lifetime:** Defer old rendered config Secret
  cleanup while `readOnlyPhase` is `ExitingReadOnly` and a cleanup Job is
  active. The cleanup Job references the config Secret via
  `QE_K8S_CONFIG_SECRET`.
- **Maximum readonly window:** 30 days. Document this operational limit.
  Pre-expiry warning (`ReadOnlyDegraded`) fires at 7 days before key
  expiration.
- **HPA bound persistence:** Store original `minReplicas`/`maxReplicas` in
  HPA annotations before rendering pinned bounds. Pinned count uses
  `HPA.status.desiredReplicas` (primary), falling back to
  `Deployment.status.replicas`, then `HPA.spec.minReplicas`. Render pinned
  desired HPA state during Recreate transitions, verify the live HPA is pinned
  before rolling Deployments, then restore from annotations after transition.
  Fall back to Kustomize-rendered values if annotations are missing.

## Scope (Quay 3.19)

`quay/quay` PR:
- `boot.py` file-based key import with `created_by` metadata and
  auto-approval, gated by `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES`
- `boot.py` readonly verification hardening
- `servicekey-expire` entrypoint mode in `quay-entrypoint.sh`
- `manage_servicekey.py` CLI with unconditional ownership check
- Keyserver GET `service` filtering fix
- New `GET /services/<service>/keys/<kid>/status` endpoint
- Readonly startup DB-write guards for `sync_database_with_config()` and
  `set_region_release()`
- JWK public-only sanitizer in `create_service_key()` and
  `replace_service_key()` for all key types (RSA, EC) — see Security Notes
- Cross-language thumbprint test fixture

`quay-operator` PR:
- CRD: `spec.readOnly`, `status.readOnlyPhase`, `status.readOnlyKeyID`,
  `status.readOnlySuppressManualConfig`, `status.readOnlyCompatibleImage`
- `ReadOnly` condition type with all reason constants listed in the Conditions
  section. Actively emits at least `ReadOnlyActive`,
  `ReadOnlyDisabled`, `ReadOnlyTransitioning`, `ReadOnlyDeferred`,
  `ReadOnlyDegraded`, `ReadOnlyCleanupFailed`, `OverrideConflict`,
  `UnsupportedVersion`, and `ManualMigrationRequired`.
- **Manual readonly detection:** Detect manual readonly lifecycle fields in
  the source config when `spec.readOnly=true` and block with
  `ManualMigrationRequired`. The rendering layer defensively strips source
  `REGISTRY_STATE` during `PreparingKey` (see Config Rendering), but this
  guard provides an earlier, explicit block with a clear admin-facing message
  rather than silently stripping config the admin intentionally set. Both
  protections should be kept. The check: if source config contains
  `REGISTRY_STATE: readonly`,
  `INSTANCE_SERVICE_KEY_KID_LOCATION`, `INSTANCE_SERVICE_KEY_LOCATION`, or
  `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES`, block with
  `ManualMigrationRequired` and message: "Source config contains manual
  readonly lifecycle fields. Remove them before enabling operator-managed
  readonly."
- `ensureReadOnlyKeySecret()` — Secret creation, immutable, volume mount
- State machine: `PreparingKey` → `EnteringReadOnly` → `ReadOnly` →
  `ExitingReadOnly` → normal
- Config rendering: inject/remove `REGISTRY_STATE`, key paths, bounded
  expiration, and `INSTANCE_SERVICE_KEY_IMPORT_FROM_FILES` per phase
- Deployment strategy: Recreate for enter/exit, Rolling otherwise
- HPA pinning/restore with annotation persistence for `quay-app` and
  `quay-mirror` during Recreate transitions. Required because the
  operator manages HPA by default and middleware intentionally avoids overriding
  Deployment replicas while HPA is managed.
- Two-step keyserver verification in `PreparingKey`
- Cleanup Job with `servicekey-expire` entrypoint, `backoffLimit: 3`,
  post-middleware construction, config Secret pinning
- Abort from `PreparingKey` and `EnteringReadOnly`
- Finalizer cleanup with 10-minute timeout
- Version gate (standard images normalized with semver and constrained to
  `>= 3.19.0-0`; custom images require annotation)
- Incompatible image rollout blocking during non-normal phases using
  `status.readOnlyCompatibleImage`
- Override conflict detection (inline only, block `valueFrom`)
- Pre-expiry warning via `/status` `expiration_date`
- `quay_operator_readonly_active` Prometheus gauge
- Basic E2E: enter, push-fails/pull-succeeds, pod restart, exit,
  push-succeeds, cleanup completes