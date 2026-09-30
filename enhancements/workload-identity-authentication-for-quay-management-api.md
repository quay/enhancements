---
title: robot-api-tokens-and-federated-exchange
authors:
  - "@Marcusk19"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-08-04
last-updated: 2026-09-30
status: provisional
see-also:
  - "https://redhat.atlassian.net/browse/PROJQUAY-11090"
  - "https://redhat.atlassian.net/browse/PROJQUAY-7803" # Original robot federation implementation
  - "https://github.com/quay/enhancements/pull/45#issuecomment-5626283676"
---

# Robot API Tokens and Federated Exchange

## 1. Purpose

Add scoped Robot API Tokens for Quay Management API access. The capability has
two credential forms:

- **Managed Robot API Tokens** are opaque, expiring `qro_` credentials created
  by an authorized signed-in human and backed by persistent lifecycle records.
- **Federated Robot API Tokens** are short-lived, Quay-signed JWTs issued by
  `/sts/token` after Quay validates an external OIDC workload identity against
  a robot federation binding.

Both forms authenticate as a configured Quay robot account, not as a human
user, bootstrap owner, or OAuth application principal.

The primary federation use case is CI/CD and Kubernetes workloads that need
Quay Management API access without storing a human password, a robot secret,
or a long-lived OAuth credential. A Federated Robot API Token can also be used
as the password in `robot-username:JWT` Basic authentication for registry
token exchange. Existing Robot Federation remains a separate registry-only
credential flow.

## 2. Change of direction

This proposal previously described an organization-owned OAuth application
that would become a new application principal and mint conventional OAuth
access tokens. That approach is **not** being pursued.

The implementation direction builds on the trust mappings from
[existing Robot Federation](https://redhat.atlassian.net/browse/PROJQUAY-7803)
without changing its registry-only token contract:

- Quay's existing non-interactive robot accounts are the workload identities.
- A federation binding maps a verified external OIDC identity to one robot.
- A dedicated STS endpoint issues a scoped Robot API JWT rather than an OAuth
  token with a hidden user or an application-only authorization model.
- The legacy Robot Federation endpoint continues to issue registry-only robot
  credentials without Management API scopes.
- The robot's existing, live Quay permissions remain the source of authority;
  token scopes can only reduce those permissions.

This avoids `BOOTSTRAP_TOKEN_OWNER`, the reserved bootstrap application, and
any login-capable user as an M2M identity. Programmatic bootstrap remains a
separate, compatible feature and is not modified by this work.

The OAuth-application-principal and org-scoped-WIF design are deferred rather
than partially implemented. If a future use case cannot be represented by a
robot identity and its explicit Quay permissions, it requires a separate
proposal.

## 3. Goals and non-goals

### Goals

- Let an external OIDC workload authenticate as an explicitly configured Quay
  robot account.
- Support Kubernetes ServiceAccount JWTs first without making the federation
  protocol Kubernetes-specific.
- Issue a Quay-signed, short-lived JWT usable with the Management API and the
  registry token flow.
- Prevent an exchanged token from granting more authority than the mapped
  robot has at request time.
- Make external identity bindings, scope ceilings, and lifecycle changes
  auditable and revocable.
- Reuse existing organization robot accounts, team roles, repository
  permissions, and robot-management UI/API surfaces.

### Non-goals

- Replacing human authentication, OAuth authorization-code flows, or existing
  programmatic bootstrap.
- Creating a new OAuth application principal or a synthetic hidden user.
- Using Kubernetes `TokenReview`, `SubjectAccessReview`, or Kubernetes RBAC to
  derive Quay permissions.
- Deriving authority from external JWT claims beyond selecting a configured
  binding.
- Supporting wildcard claims, refresh tokens, or arbitrary delegation chains
  in the initial release.
- Implicitly granting a personal namespace's permissions to a personal robot.
- Support for fine-granularity in token binding

## 4. Robot account model

A robot account is the effective Quay identity for a workload. Robot accounts
are non-human and cannot use an interactive Quay login. They already appear in
Quay's authorization model as robot `User` rows, which lets existing permission
checks, audit attribution, enabled/disabled state, organization membership, and
repository/team roles apply without impersonating a person.

The recommended workload identity is an **organization robot**. An
organization administrator creates a dedicated robot, assigns only the needed
team and repository roles, and attaches federation bindings to that robot. A
robot can therefore act only where it has normal Quay access. For example, a
CI robot can receive a Creator team role and create repositories in that
organization, while a separate release robot has only repository write access.

Personal robots may use the same federation mechanism where supported, but
are not a substitute for a user's own API token and do not inherit the user's
personal namespace permissions. Workloads needing personal-user authority
should use a separately designed user API-token capability, not a robot
permission expansion.

This is a deployment-wide federation capability with per-robot bindings. It
is not an instance-wide shared service principal: two workloads mapped to two
robots remain separate Quay identities with separate permissions and audit
records.

## 5. Authorization model

A federation binding configures the external identity, allowed OIDC audiences,
and maximum Quay API scope set for one robot. This extends the existing robot
federation binding with API-scope and audience controls.

A workload may request a narrower scope string during exchange. Omitting the
scope uses the binding's full API-scope ceiling. A request containing any scope
outside that ceiling is rejected rather than silently intersected:

```text
issued JWT scopes = requested scopes, only when requested scopes ⊆ binding api_scopes
runtime authority = issued JWT scopes ∩ robot live Quay permissions
```

The second intersection is required on every Quay request. Changing the
robot's team/repository roles, disabling it, or deleting it immediately
reduces or removes its usable authority even when an issued JWT has not yet
expired. Token scopes are a ceiling, never an independent privilege grant.

`direct_user_login` is never issued to a Robot API JWT. The feature-gated
`super:user` scope requires both deliberate issuance by a superuser and a
robot that remains a live superuser at use time. Normal CI organization
creation uses `user:admin`; it does not require `super:user` unless a
particular deployment has enabled a superuser-only organization-creation
policy.

## 6. Federation exchange

### 6.1 Legacy registry-only endpoint

Existing Robot Federation retains its original contract. The workload presents
its external JWT using the mapped robot username and HTTP Basic authentication:

```http
GET /oauth2/federation/robot/token
Authorization: Basic base64("acme+ci:EXTERNAL_OIDC_JWT")
```

After validating the external JWT and binding, Quay returns a temporary
registry-only robot credential:

```json
{
  "token": "<quay-signed-registry-only-robot-jwt>"
}
```

This endpoint never adds `api_scopes`, even when the binding contains an API
scope ceiling or the request supplies a `scope` query parameter. Its response
cannot be used as a Management API Bearer credential. This preserves existing
Robot Federation behavior independently from Federated Robot API Token
exchange.

### 6.2 RFC 8693-compatible STS endpoint

Quay provides a standards-oriented interface for Federated Robot API Tokens
using OAuth 2.0 Token Exchange
([RFC 8693](https://www.rfc-editor.org/rfc/rfc8693)). It is a separate,
additive interface; it does not change the legacy registry-only endpoint or
Quay's existing authorization-code endpoints.

```http
POST /sts/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<EXTERNAL_OIDC_JWT>
&subject_token_type=urn:ietf:params:oauth:token-type:jwt
&resource=urn:quay:robot:acme+ci
&scope=repo:read
&requested_token_type=urn:ietf:params:oauth:token-type:access_token
```

`grant_type`, `subject_token`, `subject_token_type`, and `resource` are
required. `scope` and `requested_token_type` are optional;
`requested_token_type`, when supplied, must be the access-token URN shown
above. The endpoint accepts only form-encoded requests.

The required resource URI has the form `urn:quay:robot:<full-robot-username>`.
It selects the target robot before external-token validation. Quay must never
infer the robot from an external JWT claim, so a valid token for one robot
cannot be exchanged for another robot's Quay credential.

On success, the endpoint returns a short-lived, Quay-signed Robot API JWT using
RFC 8693 response fields:

```json
{
  "access_token": "<quay-signed-robot-api-jwt>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "repo:read"
}
```

The endpoint does not issue refresh tokens or create persistent token-lifecycle
records. The JWT includes the robot subject, issued `api_scopes`, and federation
binding ID and version, and has a fixed one-hour lifetime in the initial
implementation. Omitting `scope` uses the binding's API-scope ceiling;
supplying one can only narrow that ceiling. A binding without `api_scopes`
cannot produce a Management API credential through STS.

The JWT is accepted as a Management API Bearer credential:

```http
Authorization: Bearer <quay-signed-robot-api-jwt>
```

It can also be used as the Basic-auth password for registry token exchange:

```http
Authorization: Basic base64("acme+ci:QUAY_SIGNED_JWT")
```

OAuth error responses are JSON with a non-secret error code. Parameter and
validation failures return HTTP 400 with `invalid_request`,
`unsupported_grant_type`, `invalid_target`, or `invalid_grant`. Rate-limit
exhaustion returns HTTP 429 with `slow_down` and `Retry-After`; inability to
reach the shared rate-limit store returns HTTP 503 with
`temporarily_unavailable`.

The endpoint accepts only form-encoded requests, limits the full request to 32
KiB, and limits `subject_token` to 24 KiB. Rate limiting is keyed by source IP
and robot in a fixed window. Errors and logs must not include the external JWT,
Quay JWT, authorization header, or decoded claims.

The route name is a Quay deployment choice rather than an RFC 8693
requirement. RFC 8693 standardizes the form parameters and token types, while
the authorization server advertises or documents its token-exchange endpoint.
`/sts/token` makes this workload-token exchange distinct from Quay's existing
OAuth authorization-code flow.

## 7. Binding storage, lifecycle, and revocation

Bindings are **not** static `config.yaml` configuration, OAuth-application
configuration, or a standalone database table. They are persisted with the
mapped robot's `FederatedLogin` record for the `quayrobot` login service, in
its `metadata_json` field. The `federation_config` array contains the robot's
bindings, including each binding's issuer, subject, API-scope ceiling, stable
ID, and version. This keeps the trust mapping colocated with the robot identity
that it authorizes, while the API/UI remains the supported configuration
surface.

Each persisted binding receives a stable ID and a version. The issued JWT
contains both values. On use, Quay verifies that the binding still exists and
that its version matches.

Consequently, editing a binding, deleting it, or deleting federation
configuration immediately invalidates all federation JWTs issued under the
previous binding version. This supplies revocation without persisting every
one-hour exchanged JWT. Robot disablement and live authorization checks remain
additional immediate revocation boundaries.

Binding configuration is managed on the robot through the existing personal
or organization robot-management resource and UI. Mutations are audit logged.
The schema includes issuer, subject, API scopes, stable binding identity, and a
non-empty audience allowlist. New or updated bindings default to `["quay"]`
when no audience is supplied. Quay requires the external JWT's `aud` claim to
intersect the configured allowlist.

Legacy persisted bindings without an audience remain temporarily usable with a
deprecation warning. They are transitional compatibility data, not the
steady-state security model.

## 8. Managed Robot API Tokens

Federated exchange is complemented by Managed Robot API Tokens created by a
signed-in human who is allowed to administer the robot.

A Managed Robot API Token is an opaque credential with the `qro_` prefix, not a
JWT. Quay stores its secret using credential hashing and returns the plaintext
value only in the create response. Subsequent list and detail responses expose
metadata but never return the secret.

The persistent record contains display name, creator, API-scope ceiling,
expiry, revocation time, and last-accessed time. Tokens default to 30 days and
are capped at 90 days. They can be listed and soft-revoked independently
through user- and organization-robot lifecycle API routes and the Robot Tokens
UI. Creating, revoking, expiring, or rotating one does not rotate the robot's
static registry credential.

Federated and Managed Robot API Tokens deliberately use different credential
formats and lifecycle mechanisms:

| Credential form | Format | Lifetime | Persistent token record | Revocation |
| --- | --- | --- | --- | --- |
| Managed Robot API Token | Opaque `qro_` credential | Default 30 days; maximum 90 days | Yes | Soft revoke the token record |
| Federated Robot API Token | Quay-signed JWT | Fixed one hour | No | Change/delete binding, disable robot, or change robot access |

Both forms authenticate as the same robot and are subject to the same
scope-plus-live-permission authorization rule. Managed credentials are valid
for Management API Bearer authentication and robot Basic authentication. They
cannot contain `direct_user_login`, and their plaintext value is not
recoverable after creation.

## 9. Security requirements

- Production issuer discovery requires HTTPS, exact issuer matching, hardened
  HTTP retrieval, bounded caching, no unsafe redirects, and SSRF-aware egress
  controls.
- Quay verifies JWT signature, expiration, issuer, and configured audience
  before selecting a binding.
- Bindings use exact external subjects in the initial release; no globs or
  regular expressions are permitted.
- Quay must not log external JWTs, Federated Robot API JWTs, or opaque Managed
  Robot API Token secrets.
- CSRF exemptions apply only after a valid scoped Robot API credential has
  authenticated, not to arbitrary robot credentials.
- Audit records identify the robot and safe credential provenance, such as a
  federation binding ID/version or managed token UUID/name. Exchange audit
  events may include safe external provenance such as issuer and subject, but
  never bearer secrets.

## 10. Compatibility and rollout

Two default-off feature flags control rollout:

- `FEATURE_ROBOT_API_TOKENS` enables Managed Robot API Token lifecycle routes
  and UI, and authentication for both Managed and Federated Robot API Tokens.
- `FEATURE_ROBOT_API_TOKEN_EXCHANGE` enables `/sts/token` issuance and its UI
  scope controls. Schema validation requires `FEATURE_ROBOT_API_TOKENS` when
  this flag is enabled.

When the parent capability is disabled, lifecycle routes are not registered
and return HTTP 404, while attempted Robot API Token authentication fails as
ordinary HTTP 401 authentication failure. Persisted managed token and binding
records are retained and become usable again when the capability is re-enabled
unless they expired, were revoked, or were invalidated independently.

Disabling only exchange stops new STS issuance and hides exchange controls. It
does not invalidate an already-issued Federated Robot API Token; that token
remains usable until expiry unless the binding changes, the robot is disabled,
its permissions change, or the parent capability is disabled.

The capability is otherwise additive. Existing static robot credentials,
registry-only federation, OAuth clients, human tokens, and programmatic
bootstrap retain their behavior. The legacy federation endpoint always issues
registry-only JWTs, including for bindings that also contain API scopes.
Documentation and UI should guide CI users toward a dedicated organization
robot with narrowly assigned roles and a short-lived exchanged token.

## 11. Test plan and acceptance criteria

### Unit and integration coverage

- Validate OIDC discovery/JWKS retrieval, signature, expiration, issuer,
  audience, and exact-subject matching.
- Verify an exchange rejects missing, malformed, expired, wrong-issuer,
  wrong-audience, or unbound JWTs.
- Verify omitted scopes use the binding ceiling and requested scopes outside
  that ceiling are rejected rather than silently reduced.
- Verify a valid JWT remains constrained by current robot permissions and is
  denied after robot disablement or permission removal.
- Verify STS-issued JWTs work for Management API Bearer authentication and
  registry Basic token exchange.
- Verify the legacy federation endpoint always issues a registry-only JWT with
  no `api_scopes`, even for a binding that has an API-scope ceiling.
- Verify `direct_user_login` cannot be issued and `super:user` remains
  feature-gated and live-robot-gated.
- Verify binding update/deletion invalidates already issued federation JWTs.
- Verify opaque `qro_` token creation, one-time secret return, expiry, listing,
  last-use tracking, active-token limits, hashed secret validation, and soft
  revocation.
- Verify exchange cannot be enabled without the parent feature flag.
- Verify disabled lifecycle and STS routes return HTTP 404 and disabled token
  authentication returns HTTP 401 without deleting persisted records.
- Verify disabling exchange stops new issuance while already-issued Federated
  Robot API Tokens remain valid until their normal expiry or an independent
  invalidation event.

### End-to-end coverage

- An organization admin configures a dedicated organization robot with a
  Kubernetes ServiceAccount binding and narrow API scopes.
- The workload exchanges a projected ServiceAccount JWT through `/sts/token`
  and creates or reads only resources allowed by both the token scope and the
  robot's roles.
- The STS-issued JWT completes a registry token exchange only for repositories
  the robot may access, while the legacy endpoint remains registry-only.
- An authorized human creates, uses, lists, and revokes an opaque Managed Robot
  API Token without exposing its secret after creation.
- A workload requesting an unbound scope, using another robot name, or using
  a changed/deleted binding is denied.
- A robot can bootstrap ownership of a newly created organization only when it
  has the necessary `user:admin` and `org:admin` scope/role combination; the
  intended human owner is then explicitly added to the owners team.
