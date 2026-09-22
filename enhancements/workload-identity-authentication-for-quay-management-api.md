---
title: robot-federation-for-quay-management-api
authors:
  - "@Marcusk19"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-08-04
last-updated: 2026-09-22
status: provisional
see-also:
  - "https://redhat.atlassian.net/browse/PROJQUAY-11090"
  - "https://github.com/quay/enhancements/pull/45#issuecomment-5626283676"
---

# Robot Federation for Quay Management API

## 1. Purpose

Enable an external workload to exchange its OIDC JWT for a short-lived,
scoped Quay Robot API JWT. The exchanged credential authenticates as a
configured Quay robot account, not as a human user, bootstrap owner, or OAuth
application principal.

The primary use case is CI/CD and Kubernetes workloads that need Quay
Management API access without storing a human password, a robot secret, or a
long-lived OAuth credential. The same issued JWT can also be used as the
password in `robot-username:JWT` Basic authentication for registry token
exchange.

## 2. Change of direction

This proposal previously described an organization-owned OAuth application
that would become a new application principal and mint conventional OAuth
access tokens. That approach is **not** being pursued.

The implementation direction is Robot Federation:

- Quay's existing non-interactive robot accounts are the workload identities.
- A federation binding maps a verified external OIDC identity to one robot.
- Quay issues its own signed Robot API JWT rather than an OAuth token with a
  hidden user or an application-only authorization model.
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

A federation binding configures the external identity and the maximum Quay API
scope set for one robot:

```yaml
robot: acme+ci
bindings:
  - issuer: "https://kubernetes.default.svc"
    subject: "system:serviceaccount:ci:tekton-pipeline"
    audiences: ["quay"]
    api_scopes: "repo:create repo:read repo:write"
```

The issuer and subject must match exactly after the external JWT has passed
OIDC discovery, signature, expiration, issuer, and audience validation.
Kubernetes ServiceAccounts are represented through their ordinary `sub` claim;
the protocol does not require Quay to call Kubernetes APIs.

A workload may request a narrower scope string during exchange. It may not
request scopes outside its binding:

```text
issued JWT scopes = requested scopes ∩ binding api_scopes
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

The workload presents its external JWT using the mapped robot username and
HTTP Basic authentication to the federation endpoint:

```http
GET /oauth2/federation/robot/token?scope=repo:create%20repo:read
Authorization: Basic base64("acme+ci:EXTERNAL_OIDC_JWT")
```

Quay validates the external JWT and its binding, then returns a Quay-signed
Robot API JWT:

```json
{
  "token": "<quay-signed-robot-api-jwt>"
}
```

The returned token includes the robot subject, its issued `api_scopes`, and
the federation binding ID and version. Federation-issued tokens have a fixed,
short lifetime (one hour in the initial implementation). They are not stored
as OAuth access-token records and Quay never stores the presented external JWT
or returns it in audit events.

A binding with no `api_scopes` preserves existing registry-only federation
behavior. A non-empty `api_scopes` claim marks the JWT as eligible for scoped
Management API Bearer authentication:

```http
Authorization: Bearer <quay-signed-robot-api-jwt>
```

For registry token exchange, callers use the same JWT as the Basic-auth
password:

```http
Authorization: Basic base64("acme+ci:QUAY_SIGNED_JWT")
```

## 7. Binding lifecycle and revocation

Each persisted binding receives a stable ID and a version. The issued JWT
contains both values. On use, Quay verifies that the binding still exists and
that its version matches.

Consequently, editing a binding, deleting it, or deleting federation
configuration immediately invalidates all federation JWTs issued under the
previous binding version. This supplies revocation without persisting every
one-hour exchanged JWT. Robot disablement and live authorization checks remain
additional immediate revocation boundaries.

Binding configuration is managed on the robot, through the existing personal
or organization robot-management resource and UI. Mutations are audit logged.
The initial schema includes issuer, subject, API scopes, and binding identity;
production Kubernetes workload federation also requires audience validation.
Audience configuration is a required completion item before this is considered
production-ready. Legacy audience-less bindings are transitional only and must
be deprecated rather than treated as the steady-state security model.

## 8. Human-created Robot API Tokens

Robot Federation is complemented, not replaced, by Robot API Tokens created by
a signed-in human who is allowed to manage that robot.

A human-created Robot API Token is also a Quay-signed Robot API JWT, but has
recorded lifecycle metadata: display name, creator, expiry, revocation time,
and last-accessed time. It defaults to 30 days and is capped at 90 days. The
owner can list and revoke its active tokens through the Robot Tokens UI/API.

Federation-issued and human-created tokens deliberately use different
lifecycle mechanisms:

| Credential source | Lifetime | Persistent token record | Revocation |
| --- | --- | --- | --- |
| Human-created Robot API Token | Default 30 days; maximum 90 days | Yes | Soft revoke the token record |
| Federated Robot API JWT | Fixed one hour | No | Change/delete binding, disable robot, or change robot access |

Both forms authenticate as the same robot and are subject to the same
scope-plus-live-permission authorization rule.

## 9. Security requirements

- Production issuer discovery requires HTTPS, exact issuer matching, hardened
  HTTP retrieval, bounded caching, no unsafe redirects, and SSRF-aware egress
  controls.
- Quay verifies JWT signature, expiration, issuer, and configured audience
  before selecting a binding.
- Bindings use exact external subjects in the initial release; no globs or
  regular expressions are permitted.
- Quay must not log the external JWT or the returned Quay JWT.
- CSRF exemptions apply only to valid scoped Robot API JWT requests, not to
  arbitrary robot credentials.
- Audit records identify the robot and safe external provenance such as issuer
  and subject; they never include bearer secrets.

Local development may allow HTTP discovery only when Quay explicitly runs in
`DEBUG` mode. This exception is not available in production.

## 10. Compatibility and rollout

- The federation capability is additive. Existing robot credentials,
  registry-only federation bindings, OAuth clients, human tokens, and
  programmatic bootstrap retain their behavior.
- Existing federation bindings without API scopes continue to issue
  registry-only JWTs and cannot call the Management API.
- API scope support, binding ID/version invalidation, and audience validation
  are rolled out behind the normal feature/configuration controls.
- Documentation and UI should guide CI users toward a dedicated organization
  robot with narrowly assigned roles and a short-lived exchanged token.

## 11. Test plan and acceptance criteria

### Unit and integration coverage

- Validate OIDC discovery/JWKS retrieval, signature, expiration, issuer,
  audience, and exact-subject matching.
- Verify an exchange rejects missing, malformed, expired, wrong-issuer,
  wrong-audience, or unbound JWTs.
- Verify requested scopes cannot exceed the binding scopes.
- Verify a valid JWT remains constrained by current robot permissions and is
  denied after robot disablement or permission removal.
- Verify Management API Bearer authentication and registry Basic token exchange
  work for scoped Robot API JWTs.
- Verify `direct_user_login` cannot be issued and `super:user` remains
  feature-gated and live-robot-gated.
- Verify binding update/deletion invalidates already issued federation JWTs.
- Verify human-created token expiry, listing, last-use tracking, active-token
  limits, and soft revocation.

### End-to-end coverage

- An organization admin configures a dedicated organization robot with a
  Kubernetes ServiceAccount binding and narrow API scopes.
- The workload exchanges a projected ServiceAccount JWT and creates or reads
  only resources allowed by both the token scope and the robot's roles.
- The same JWT completes a registry token exchange only for repositories the
  robot may access.
- A workload requesting an unbound scope, using another robot name, or using
  a changed/deleted binding is denied.
- A robot can bootstrap ownership of a newly created organization only when it
  has the necessary `user:admin` and `org:admin` scope/role combination; the
  intended human owner is then explicitly added to the owners team.

The shipping bar is:

> A bound external workload can obtain a short-lived Quay-signed credential
> for one non-interactive robot account. It can perform only actions that are
> permitted by both its issued scopes and the robot's current Quay permissions;
> humans, bootstrap identities, and unrelated robots are never impersonated.
