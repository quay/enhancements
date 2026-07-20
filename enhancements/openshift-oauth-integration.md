---
title: openshift-oauth-integration
authors:
  - "@bpratt"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-02-16
last-updated: 2026-02-16
status: implementable
see-also:
  - "/enhancements/teamsync-with-oidc.md"
  - "/enhancements/kubernetes-sa-oidc-auth.md"
---

# OpenShift OAuth Integration

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

1. **Email handling.** OpenShift does not expose user email addresses through its User API (`/apis/user.openshift.io/v1/users/~` returns `null` for email). Quay features that depend on email (notifications, account recovery) will be unavailable for OpenShift-authenticated users unless an external identity provider (e.g., LDAP backing the OAuth server) populates email through a custom claim. Should we require a fallback email prompt on first login, or accept that email-dependent features are degraded?

2. **Opaque token caching.** OpenShift access tokens are opaque (not JWTs), so every bearer-token API call requires a round-trip to the OpenShift User API for validation. Should we implement a short-TTL cache (keyed on token hash) to reduce API server load, or is the simplicity of always-validate preferred for the initial release?

3. **Service account RBAC scope for team sync.** Background team sync requires a service account token with `get` on `groups.user.openshift.io` and `get`/`list` on `users.user.openshift.io`. These are read-only but cluster-scoped. Is this acceptable, or should we support namespace-scoped group enumeration via a different API?

4. **Out-of-cluster deployment.** When Quay runs outside the OpenShift cluster, `KUBERNETES_SERVICE_HOST` is not set and the CA bundle is not mounted. The admin must manually configure `OPENSHIFT_API_URL` and manage TLS trust. Should we document this as a supported topology or explicitly require in-cluster deployment?

5. **Multi-cluster federation.** A single Quay instance may serve multiple OpenShift clusters. The current design supports one `OPENSHIFT_LOGIN_CONFIG`. Should we support multiple OpenShift providers (e.g., `OPENSHIFT_CLUSTER1_LOGIN_CONFIG`, `OPENSHIFT_CLUSTER2_LOGIN_CONFIG`), or defer this to a future enhancement?

## Summary

This enhancement adds native OpenShift OAuth 2.0 as a first-class authentication provider for Quay. Users on OpenShift clusters can log in to Quay using their OpenShift credentials, and Quay teams can be synchronized with OpenShift groups.

OpenShift's OAuth server is **not standard OIDC**. It differs in three critical ways that prevent the existing generic OIDC login from working:

| Aspect | Standard OIDC | OpenShift OAuth |
|--------|--------------|-----------------|
| **Discovery** | `/.well-known/openid-configuration` (OIDC Discovery) | `/.well-known/oauth-authorization-server` (RFC 8414) |
| **Token format** | JWT `id_token` with claims | Opaque `access_token` only, no `id_token` |
| **User info** | `/userinfo` endpoint (OIDC Core 5.3) | No `/userinfo`; must call `/apis/user.openshift.io/v1/users/~` |
| **Group membership** | `groups` claim in `id_token` or `/userinfo` | Separate Groups API: `/apis/user.openshift.io/v1/groups/{name}` |

A dedicated `OpenShiftOAuthService` subclass handles these protocol differences while reusing the existing OIDC infrastructure for the authorization code flow, token exchange, and PKCE.

## Motivation

OpenShift deployments that want SSO between the cluster and Quay currently have two options: deploy an identity broker (Keycloak/RHSSO) in front of OpenShift's OAuth server, or use LDAP directly. Both add infrastructure complexity, configuration burden, and additional failure domains.

Native OpenShift OAuth integration eliminates the need for an intermediary identity provider on OpenShift deployments, providing direct SSO with the platform's built-in OAuth server.

### Goals

- Allow OpenShift users to log in to Quay using their cluster credentials via OAuth 2.0 authorization code flow
- Support team synchronization between Quay teams and OpenShift groups
- Auto-detect in-cluster configuration (API server URL, CA certificate, service account token) when Quay runs as a pod on OpenShift
- Gracefully fall back to standard OIDC discovery if RFC 8414 metadata is not available (e.g., OpenShift behind Keycloak)
- Support PKCE for public OAuth clients

### Non-Goals

- **CLI token authentication** (`oc login` style bearer tokens for `podman`/`docker` CLI). This is a separate feature that requires Docker v2 token auth integration.
- **RBAC mapping.** Mapping OpenShift RBAC roles to Quay permissions (e.g., namespace admin -> org admin). Quay manages its own permissions model.
- **Multi-cluster support.** Authenticating against multiple OpenShift clusters from a single Quay instance. The current design supports one `OPENSHIFT_LOGIN_CONFIG`.
- **Email features.** OpenShift does not provide user email; features requiring email (notifications, recovery) are not addressed.
- **User provisioning.** Automatically creating Quay organizations or repositories based on OpenShift namespaces.

## Proposal

### Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         OpenShift Cluster                                │
│                                                                          │
│  ┌──────────────────┐    ┌──────────────────┐    ┌────────────────────┐  │
│  │  OAuth Server     │    │  API Server       │    │  Quay Pod          │  │
│  │  (oauth-openshift │    │  (api.cluster)    │    │                    │  │
│  │   .apps.cluster)  │    │                   │    │  ┌──────────────┐  │  │
│  │                   │    │  /apis/user.       │    │  │ OpenShift    │  │  │
│  │  /authorize       │    │    openshift.io/   │    │  │ OAuthService │  │  │
│  │  /token           │    │    v1/users/~      │    │  │              │  │  │
│  │  /.well-known/    │    │    v1/groups/      │    │  └──────┬───────┘  │  │
│  │   oauth-authz-    │    │                   │    │         │          │  │
│  │   server          │    │                   │    │         │          │  │
│  └────────┬──────────┘    └────────┬──────────┘    └─────────┼──────────┘  │
│           │                        │                         │             │
│           │  ①  Authorization      │  ③  GET /users/~        │             │
│           │     Code Grant         │     (access_token)      │             │
│           │                        │                         │             │
│           │  ②  Token Exchange     │  ④  GET /groups/{name}  │             │
│           │     (code → token)     │     (SA token, for      │             │
│           │                        │      team sync)         │             │
│           └────────────────────────┴─────────────────────────┘             │
│                                                                          │
│  ┌──────────────────┐                                                     │
│  │  User Browser     │  ←── Redirect-based login flow                     │
│  └──────────────────┘                                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

### Login Flow

1. **User clicks "Log in with OpenShift"** on the Quay login page
2. **Quay redirects** to OpenShift OAuth server's `/authorize` endpoint with `response_type=code`, `client_id`, `redirect_uri`, and scopes (`user:info`)
3. **User authenticates** with OpenShift (htpasswd, LDAP, GitHub IdP, etc.)
4. **OpenShift redirects back** to Quay with an authorization code
5. **Quay exchanges the code** for an opaque `access_token` at the `/token` endpoint (no `id_token` is returned)
6. **Quay calls the User API** (`/apis/user.openshift.io/v1/users/~`) with the `access_token` to get username, UID, and group membership
7. **Quay creates or updates** the federated user record and logs the user in

### Team Sync Flow

1. **Admin configures team sync** by linking a Quay team to an OpenShift group name
2. **Team sync worker** runs on a configurable interval
3. **Worker reads the in-cluster service account token** from `/var/run/secrets/kubernetes.io/serviceaccount/token`
4. **Worker calls the Groups API** (`/apis/user.openshift.io/v1/groups/{name}`) to get the list of usernames
5. **Worker fetches each user's details** from the Users API (`/apis/user.openshift.io/v1/users/{username}`)
6. **Worker synchronizes** Quay team membership to match the OpenShift group

### User Stories

#### Story 1: OpenShift SSO Login

As a developer on an OpenShift cluster, I want to log in to Quay using my OpenShift credentials so that I have a single identity across the platform without managing a separate Quay account.

**Acceptance criteria:**
- Login page shows an "OpenShift" button when `FEATURE_OPENSHIFT_LOGIN` is enabled
- Clicking the button redirects to the OpenShift OAuth server
- After authenticating with OpenShift, I am redirected back to Quay and logged in
- My Quay username matches my OpenShift username

#### Story 2: Team Synchronization with OpenShift Groups

As a cluster administrator, I want Quay teams to be automatically synchronized with OpenShift groups so that when I add a user to an OpenShift group, they automatically get the corresponding Quay team permissions.

**Acceptance criteria:**
- I can link a Quay team to an OpenShift group name in the team settings
- Quay validates that the group exists in OpenShift before accepting the configuration
- Team membership is updated on the configured sync interval
- Users removed from the OpenShift group are removed from the Quay team

#### Story 3: Zero-Configuration In-Cluster Deployment

As an operator deploying Quay on OpenShift, I want the OpenShift API URL and TLS configuration to be auto-detected so that I only need to configure the OAuth client credentials.

**Acceptance criteria:**
- When running as a pod on OpenShift, Quay auto-detects the API server URL from `KUBERNETES_SERVICE_HOST`
- TLS verification uses the mounted CA bundle at `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
- Only `CLIENT_ID` and `OIDC_SERVER` are required in the configuration

### Implementation Details

The implementation introduces two new classes that extend existing OIDC infrastructure:

| Component | Class | Base Class | Purpose |
|-----------|-------|------------|---------|
| Login service | `OpenShiftOAuthService` | `OIDCLoginService` | Handles OAuth flow, discovery, token exchange, user info |
| Users backend | `OpenShiftUsers` | `OIDCUsers` | Provides `iterate_group_members()` for background team sync |

**Key behavioral differences from standard OIDC:**

| Behavior | Standard OIDC (`OIDCLoginService`) | OpenShift (`OpenShiftOAuthService`) |
|----------|-----------------------------------|-------------------------------------|
| Discovery URL | `/.well-known/openid-configuration` | `/.well-known/oauth-authorization-server` (RFC 8414) |
| Discovery fallback | None | Falls back to standard OIDC discovery if RFC 8414 returns 404 |
| Token response | `id_token` (JWT) + `access_token` | `access_token` only (opaque) |
| User info source | Decode `id_token` claims or call `/userinfo` | Call `/apis/user.openshift.io/v1/users/~` with `access_token` |
| Group source | `groups` claim in `id_token` or `/userinfo` | Response field from User API, or Groups API for team sync |
| OAuth scopes | `openid`, `profile`, `email` | `user:info` |
| Bearer token validation | Decode JWT, verify signature against JWKS | Call User API (opaque token introspection) |
| Email availability | Yes (from `email` claim) | No (OpenShift User API does not expose email) |

**Configuration schema (`OPENSHIFT_LOGIN_CONFIG`):**

```yaml
FEATURE_OPENSHIFT_LOGIN: true
AUTHENTICATION_TYPE: OpenShift

OPENSHIFT_LOGIN_CONFIG:
  # Required
  OIDC_SERVER: "https://oauth-openshift.apps.cluster.example.com"
  CLIENT_ID: "quay-registry"

  # Optional (auto-detected in-cluster)
  OPENSHIFT_API_URL: "https://api.cluster.example.com:6443"

  # Optional with defaults
  CLIENT_SECRET: ""                    # Not needed if PUBLIC_CLIENT is true
  PUBLIC_CLIENT: true                  # Use PKCE instead of client secret
  SERVICE_NAME: "OpenShift"            # Login button label
  SERVICE_ICON: "fa-openshift"         # Login button icon
  LOGIN_SCOPES: ["user:info"]          # OAuth scopes
  USE_PKCE: true                       # PKCE for public clients
  PREFERRED_GROUP_CLAIM_NAME: "groups" # Group field in API response
  SERVICE_ACCOUNT_TOKEN_PATH: "/var/run/secrets/kubernetes.io/serviceaccount/token"
```

**Required OpenShift resources:**

```yaml
apiVersion: oauth.openshift.io/v1
kind: OAuthClient
metadata:
  name: quay-registry
redirectURIs:
  - "https://quay.apps.cluster.example.com/oauth2/openshift/callback"
grantMethod: auto  # or "prompt" for explicit user consent
```

**Required RBAC for team sync (service account):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: quay-group-reader
rules:
  - apiGroups: ["user.openshift.io"]
    resources: ["groups"]
    verbs: ["get"]
  - apiGroups: ["user.openshift.io"]
    resources: ["users"]
    verbs: ["get", "list"]
```

### Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| OpenShift API server unavailability blocks login | Users cannot authenticate | Low (HA control plane) | Existing sessions remain valid; document that login depends on API server health |
| Opaque token validation adds latency per API call | Increased response time for bearer-token authenticated requests | Medium | Implement token-hash-keyed cache with short TTL (open question 2) |
| Service account token expiration during long sync | Team sync fails mid-operation | Low | Kubernetes rotates projected tokens automatically; worker re-reads token file on each run |
| User search lists all cluster users | Scalability issue on large clusters (1000+ users) | Medium | `query_users()` fetches all users and filters locally; document as known limitation, consider pagination in future |
| Breaking changes in OpenShift User/Group API | Integration stops working after cluster upgrade | Low | Pin to `v1` API version; API has been stable since OpenShift 4.0 |

## Design Details

### Test Plan

**Unit tests:**
- `OpenShiftOAuthService` discovery with RFC 8414 metadata (mock HTTP)
- Fallback to standard OIDC discovery when RFC 8414 returns 404
- `exchange_code_for_tokens` returns `(None, access_token)` (no `id_token`)
- `exchange_code_for_login` calls User API and extracts username/UID/groups
- `validate_opaque_token` calls User API and returns user info
- `OpenShiftUsers.iterate_group_members` returns correct `UserInformation` objects
- `OpenShiftUsers.check_group_lookup_args` handles 200, 404, and 403 responses
- `OpenShiftUsers.query_users` filters results by query string
- `_get_openshift_api_url` auto-detection from `KUBERNETES_SERVICE_HOST`

**Integration tests:**
- Full OAuth authorization code flow with mocked OpenShift OAuth server
- Opaque token bearer auth through `validate_bearer_auth` -> `validate_openshift_opaque_token`
- Team sync worker processes OpenShift groups and updates Quay team membership
- `AUTHENTICATION_TYPE: OpenShift` wires up `OpenShiftUsers` backend via `get_users_handler`

**Manual E2E validation:**
- Deploy Quay on an OpenShift cluster with `FEATURE_OPENSHIFT_LOGIN: true`
- Log in via OpenShift OAuth and verify user creation
- Configure team sync with an OpenShift group and verify membership propagation
- Remove a user from the OpenShift group and verify they are removed from the Quay team
- Verify login works after cluster upgrade (minor version)

### Graduation Criteria

#### Dev Preview -> Tech Preview

- All unit and integration tests passing
- Login flow works end-to-end on OpenShift 4.14+
- Team sync works with OpenShift groups
- Documentation covers configuration and required RBAC
- Feature gated behind `FEATURE_OPENSHIFT_LOGIN: false` (opt-in)

#### Tech Preview -> GA

- Tested across OpenShift 4.14, 4.15, 4.16, and 4.17
- Upgrade/downgrade testing from Quay version without OpenShift support
- Performance characterization of opaque token validation under load
- User search scalability tested with 500+ cluster users
- Token caching strategy finalized (if implemented)
- Available by default on OpenShift deployments via quay-operator

### Upgrade / Downgrade Strategy

**Upgrade (Quay without OpenShift OAuth -> Quay with OpenShift OAuth):**
- No database migrations required. OpenShift users are stored in existing `FederatedLogin` table with `service_name = "openshift"`.
- Existing OIDC or LDAP users are unaffected. The feature is opt-in via `FEATURE_OPENSHIFT_LOGIN` and `AUTHENTICATION_TYPE: OpenShift`.
- Admins must create the `OAuthClient` CR in OpenShift and configure `OPENSHIFT_LOGIN_CONFIG` in Quay before enabling.

**Downgrade (Quay with OpenShift OAuth -> Quay without OpenShift OAuth):**
- Set `FEATURE_OPENSHIFT_LOGIN: false` and change `AUTHENTICATION_TYPE` back to previous value.
- `FederatedLogin` records with `service_name = "openshift"` become orphaned but do not cause errors. Users will need to re-authenticate with the new auth provider.
- Team sync entries referencing OpenShift groups stop updating but do not cause failures; the sync worker skips groups it cannot resolve.

### Version Skew Strategy

- **OpenShift version:** The implementation targets the `user.openshift.io/v1` API, which has been stable and unchanged since OpenShift 4.0. The `/.well-known/oauth-authorization-server` endpoint (RFC 8414) is available on all OpenShift 4.x releases. OpenShift 3.x is not supported.
- **Quay operator version:** The operator does not need changes for this feature. Configuration is purely in `config.yaml`. A future operator enhancement could auto-create the `OAuthClient` CR and inject configuration.
- **Multiple Quay replicas:** All replicas use the same configuration. Opaque token validation is stateless (each replica calls the User API independently). If token caching is added, each replica maintains its own cache (no cross-replica coordination required).

## Implementation History

- 2026-02-16: Initial proposal
- 2026-02-16: Implementation on `openshift-oauth` branch (`e48ae973e`)

## Drawbacks

1. **Increased auth code surface area.** Adding `OpenShiftOAuthService` and `OpenShiftUsers` introduces ~780 lines of new code in the authentication path. This code must be maintained alongside the existing OIDC, LDAP, Keystone, and Database auth backends.

2. **Tight coupling to API server availability.** Unlike JWT-based OIDC (where tokens can be validated offline using cached JWKS), opaque token validation requires a live connection to the OpenShift API server. If the API server is unreachable, new logins and bearer-token API calls will fail.

3. **No email support.** OpenShift's User API does not expose email addresses. Quay features that depend on email (notifications, account recovery, Gravatar) will not work for OpenShift-authenticated users without supplementary configuration.

4. **Team sync RBAC prerequisites.** Background team sync requires a service account with cluster-scoped `get` on `groups.user.openshift.io` and `get`/`list` on `users.user.openshift.io`. While read-only, cluster-scoped roles may require approval from cluster administrators in locked-down environments.

5. **User search scalability.** OpenShift does not provide a user search API. The implementation lists all users and filters locally (`query_users()`), which may be slow on clusters with thousands of users.

## Alternatives

### Alternative 1: Generic OIDC (No Code Changes)

Use the existing `<PREFIX>_LOGIN_CONFIG` OIDC support to connect to OpenShift's OAuth server.

**Why it doesn't work:**

- **Discovery fails.** OpenShift serves `/.well-known/oauth-authorization-server` (RFC 8414), not `/.well-known/openid-configuration`. The existing `OIDCLoginService` only tries OIDC discovery and raises `DiscoveryFailureException`.
- **No `id_token`.** OpenShift's `/token` endpoint returns only an opaque `access_token`. The OIDC code path expects a JWT `id_token` to extract `sub`, `preferred_username`, and `email` claims.
- **No `/userinfo` endpoint.** The fallback for missing `id_token` claims is `/userinfo`, which OpenShift does not implement.
- **No group support.** Even if discovery and user info worked, OpenShift does not put group membership in token claims. Team sync requires the Groups API.

**Pros:**
- Zero code changes

**Cons:**
- Does not work. All four protocol differences are blocking.

### Alternative 2: TokenReview API

Use the Kubernetes `TokenReview` API to validate OpenShift tokens server-side.

**Pros:**
- Validates any token type (opaque or JWT)
- Works with all OpenShift authentication backends
- Direct API server validation (no discovery needed)

**Cons:**
- Does not solve the login flow. `TokenReview` validates existing tokens but cannot initiate an OAuth authorization code grant. Users would need to obtain tokens out-of-band (e.g., `oc login`).
- Does not provide group enumeration. `TokenReview` returns the user's identity but not their group memberships. Team sync would still require the Groups API.
- Requires `create` on `tokenreviews` (a powerful RBAC permission) rather than read-only group/user access.
- Every API call requires a round-trip to the API server (same as opaque token validation, but with a heavier API).

### Alternative 3: Native OpenShift OAuth (This Proposal)

Extend Quay's OIDC infrastructure with an OpenShift-specific subclass that handles the protocol differences.

**Pros:**
- Handles all four protocol differences (discovery, token format, user info, groups)
- Minimal RBAC requirements (read-only group/user access for team sync; no RBAC needed for login)
- In-cluster auto-detection eliminates manual API URL and TLS configuration
- Graceful OIDC fallback: if RFC 8414 discovery fails, falls back to standard OIDC discovery (supports OpenShift behind Keycloak)
- Reuses existing OIDC code (authorization code flow, token exchange, PKCE, login manager)
- Plugs into existing team sync worker without modifications

**Cons:**
- New code to maintain (~780 lines across `OpenShiftOAuthService` and `OpenShiftUsers`)
- Opaque token validation requires API server connectivity
- OpenShift-specific logic in the auth path (not generalizable to other OAuth 2.0-only providers)

### Alternative 4: Keycloak / RHSSO Identity Broker

Deploy Keycloak or Red Hat SSO as an identity broker between OpenShift and Quay. Keycloak federates with OpenShift's OAuth server and presents a standard OIDC interface to Quay.

**Pros:**
- Quay uses standard OIDC with no code changes
- Keycloak provides `/userinfo`, JWT `id_token`, and group claims
- Supports multi-cluster federation (multiple OpenShift IdP backends in one Keycloak realm)
- Rich identity management features (attribute mapping, protocol mappers, user federation)
- Email can be sourced from the backing identity provider (LDAP, etc.)

**Cons:**
- Adds significant infrastructure: Keycloak deployment, database, TLS certificates, backup/restore
- Complex configuration: OpenShift IdP in Keycloak, client registration, group mappers, scope configuration
- Additional failure domain: if Keycloak is down, all Quay authentication fails
- Latency: login flow goes through an additional redirect hop (browser -> Keycloak -> OpenShift -> Keycloak -> Quay)
- Operational burden: Keycloak upgrades, security patches, realm configuration management
- Not appropriate as the default/recommended path for simple OpenShift deployments where a direct integration suffices

**Recommendation:** Keycloak remains a viable option for organizations that already run it or need advanced identity features (multi-cluster, email mapping, protocol translation). It should not be the only supported path for OpenShift SSO, as it imposes unnecessary complexity on deployments that only need basic login and team sync.
