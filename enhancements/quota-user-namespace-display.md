---
title: Quota Totals Display for User Namespaces
authors:
  - TBD
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2023-08-07
last-updated: 2023-08-07
status: implemented
see-also:
  - "/enhancements/quota-management.md"
---

# Quota Totals Display for User Namespaces

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Quay's quota management feature (`FEATURE_QUOTA_MANAGEMENT`) allows superusers to set and enforce storage limits per namespace. The initial implementation (see [quota-management.md](quota-management.md)) focused on organization namespaces. However, every Quay user also has a personal namespace (named after their username) that can hold repositories and accumulate storage. Before this enhancement, quota consumption was not surfaced in the UI for user (personal) namespaces, leaving users and administrators without visibility into personal namespace storage usage even when a quota was configured via the superuser API.

This enhancement extends the quota display to user namespaces so that quota consumption, limits, and backfill status are visible in the same way they are for organization namespaces.

## Motivation

Quay's data model treats user namespaces and organization namespaces uniformly at the database level — both are represented by the `User` model, with `organization=False` distinguishing personal accounts. Quotas can already be configured for user namespaces through the superuser API (`/api/v1/superuser/users/<namespace>/quota`). Despite this, the UI showed no quota information on the user namespace view, creating an inconsistency:

- A superuser could configure a quota on a user namespace.
- The quota would be enforced on push (if `FEATURE_VERIFY_QUOTA` is enabled).
- But neither the user nor the superuser could see the current consumption in the UI.

This gap reduces operational visibility and makes it harder for administrators to audit storage usage across all namespace types.

### Goals

- Display total quota consumed (bytes and percentage) for user namespaces in the Quay UI when `FEATURE_QUOTA_MANAGEMENT` is enabled.
- Show the configured quota limit alongside the consumption figure.
- Surface the quota backfill status (queued or running) when `QUOTA_BACKFILL` is enabled, consistent with how it is shown for organization namespaces.
- Ensure the superuser API correctly returns quota data for user namespaces via `SuperUserUserQuotaList`.

### Non-Goals

- This enhancement does not introduce new quota enforcement behaviour; enforcement is handled by `FEATURE_VERIFY_QUOTA` and the push critical path (covered in the original quota management enhancement).
- This enhancement does not add quota configuration UI for non-superusers on user namespaces; quota limits for user namespaces remain superuser-only.
- Changes to the auto-pruning or garbage collection pipelines are out of scope.

## Proposal

### User Stories

#### Story 1 — User views their personal namespace quota

As a Quay user, when I navigate to my personal namespace page, I want to see how much storage I have consumed and what my configured quota limit is (if one exists), so I can make informed decisions about pushing new images or deleting old ones.

#### Story 2 — Superuser audits user namespace storage

As a Quay superuser, when I set a quota on a user namespace via the API, I want the user namespace view to reflect that quota and its current consumption, so I can verify enforcement is working and users can see the constraint.

### Implementation Details

#### API Layer — `org_view` in `endpoints/api/organization.py`

The `org_view` function constructs the response payload for both organization and user namespace views. When `features.QUOTA_MANAGEMENT` and `features.EDIT_QUOTA` are enabled, it now:

1. Fetches the namespace quota list via `model.namespacequota.get_namespace_quota_list(namespace.username)`.
2. Fetches the quota report (consumed bytes and limit) via the quota model.
3. Includes `"quotas"` and `"quota_report"` keys in the response regardless of whether the namespace belongs to an organization or an individual user.

This works without special-casing because the underlying `namespacequota` model functions accept any `User` entity (both `organization=True` and `organization=False`).

#### Superuser API — `SuperUserUserQuotaList` in `endpoints/api/superuser.py`

The `SuperUserUserQuotaList` resource exposes `GET`, `POST` endpoints at `/api/v1/superuser/users/<namespace>/quota`. The `get` method resolves the namespace to a `User` entity via `user.get_user_or_org(namespace)` and calls `namespacequota.get_namespace_quota_list(namespace_user.username)`, making it namespace-type-agnostic. This ensures the API correctly returns quota data for personal user namespaces as well as organization namespaces.

#### UI Layer — `static/partials/org-view.html`

The org-view template is shared between organization and user namespace pages. The following UI elements are conditionally rendered when `Features.QUOTA_MANAGEMENT` is enabled:

- **Total Quota Consumed**: displays the consumed storage in human-readable units.
- **Percentage consumed**: shown when a quota limit is configured (`Features.EDIT_QUOTA`).
- **Configured quota limit**: the limit set by the superuser.
- **Backfill status indicator**: shown when `QUOTA_BACKFILL` is enabled and the namespace size calculation is still queued or running.

Because the template is shared, enabling quota display for user namespaces requires no template duplication — only ensuring the backend passes the correct quota data for user namespace requests.

#### Database Models

No new database models or migrations are required. The existing models are sufficient:

| Model | Purpose |
|---|---|
| `UserOrganizationQuota` | Stores the quota limit in bytes for a namespace (user or org) |
| `QuotaNamespaceSize` | Stores consumed bytes and backfill status for a namespace |
| `QuotaLimits` | Links a quota to a `QuotaType` (Warning / Reject) with a percentage threshold |

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Quota data absent for user namespace causes UI errors | The template conditionally renders quota elements only when `quota_report` is populated; missing data degrades gracefully to no display |
| Performance: additional quota query per user namespace page load | The quota queries are lightweight point lookups by namespace primary key; no paginated scans involved |
| Superuser quota API returns stale data during backfill | The `QuotaNamespaceSize.backfill_status` field is surfaced in the UI so users can see when figures are still being computed |

## Design Details

### Feature Flags

| Flag | Effect |
|---|---|
| `FEATURE_QUOTA_MANAGEMENT` | Master switch; enables quota reporting in the UI |
| `FEATURE_EDIT_QUOTA` | Enables display of the configured limit and percentage consumed |
| `FEATURE_VERIFY_QUOTA` | Enforces limits on push (separate from display; not changed here) |
| `QUOTA_BACKFILL` | Enables the background worker that computes sizes for pre-existing images |

### API Design

No new endpoints are introduced. The existing endpoints below are confirmed to work for user namespaces:

**Superuser quota management (user namespace)**

```
GET    /api/v1/superuser/users/<namespace>/quota
POST   /api/v1/superuser/users/<namespace>/quota
PUT    /api/v1/superuser/users/<namespace>/quota/<quota_id>
DELETE /api/v1/superuser/users/<namespace>/quota/<quota_id>
```

All endpoints require superuser privileges. The `<namespace>` parameter accepts both organization names and individual usernames.

**Namespace view (includes quota report)**

```
GET /api/v1/organization/<orgname>
```

Returns `quotas` and `quota_report` fields when `FEATURE_QUOTA_MANAGEMENT` is enabled, for both organization and user namespaces.

### Test Plan

- **Unit tests**: Verify that `org_view` returns `quota_report` data when called with a user namespace (not an organization) and `FEATURE_QUOTA_MANAGEMENT` is enabled.
- **Unit tests**: Verify that `SuperUserUserQuotaList.get()` returns the correct quota list for a user namespace.
- **Integration tests**: Configure a quota on a user namespace via the superuser API and confirm the namespace view response includes the quota report.
- **UI tests (Cypress/Playwright)**: Navigate to a user namespace page with a configured quota and assert the "Total Quota Consumed" element is visible and shows the correct value.

### Upgrade / Downgrade Strategy

- **Upgrade**: No migration required. Existing `UserOrganizationQuota` and `QuotaNamespaceSize` rows for user namespaces are read by the updated code paths automatically.
- **Downgrade**: Reverting the code change restores the previous behaviour (no quota display on user namespace pages). No data loss occurs; quota rows for user namespaces remain in the database.

## Implementation History

- 2023-08-07 Initial implementation (PROJQUAY-5850) — quota totals display extended to user namespaces; committed in quay/quay@65c1829b
- 2023-08-30 Follow-up refinement (quay/quay@7f54e765)

## Drawbacks

- The UI change is additive and low-risk, but it ties the user namespace view more tightly to the organization view template. Future divergence of the two views may require template separation.

## Alternatives

- **Separate user namespace view template**: Would allow independent evolution of user and org views, but adds duplication for what is currently identical quota display logic.
- **Quota display only for organizations**: Preserves the status quo but leaves users without visibility into limits applied to their personal namespaces.
