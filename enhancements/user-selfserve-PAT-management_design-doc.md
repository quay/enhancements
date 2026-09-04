# PROJQUAY-6385 — User Self-Service API Token Management
## Enhancement Design Document — Phase 3

**Jira:** [PROJQUAY-6385](https://redhat.atlassian.net/browse/PROJQUAY-6385)  
**Dependencies:** [PROJQUAY-12058](https://redhat.atlassian.net/browse/PROJQUAY-12058) (Phase 1 GA), [PROJQUAY-10436](https://redhat.atlassian.net/browse/PROJQUAY-10436) (Phase 1), [PROJQUAY-9755](https://redhat.atlassian.net/browse/PROJQUAY-9755) (Phase 2)

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Acceptance Criteria](#2-acceptance-criteria)
3. [Solution Proposal](#3-solution-proposal)
4. [Key Design Decisions](#4-key-design-decisions)
5. [User Journeys](#5-user-journeys)
6. [Limitations and Clarifications Required](#6-limitations-and-clarifications-required)
7. [Risk of making application_id nullable and mitigation plan](#7-risk-of-making-application_id-nullable-and-mitigation-plan)
8. [Appendix A: Reference Material](#appendix-a-reference-material)

---

## 1. Problem Statement

### 1.1 Executive Summary

Red Hat Quay's current token management model poses a significant supply-chain security risk on quay.io. Users cannot create fine-grained, repository-scoped API tokens. Instead, all tokens are blanket-scoped — a single `repo:read` token grants read access to **every repository** across **every organization** the user belongs to. Combined with the fact that many users select broad scopes like `org:admin` for CI/CD convenience, this means a single compromised token on a developer's laptop could grant an attacker full administrative access to critical infrastructure images (including OCP release images in the `openshift` organization on quay.io).

### 1.2 The Three Core Problems

#### Problem 1: Tokens Are Too Powerful (No Per-Resource Scoping)

Today's OAuth scopes are **namespace-wide**. When a user selects a scope during token authorization, that scope applies globally:

| Scope Selected | What the User Expects | What Actually Happens |
|---|---|---|
| `repo:read` | "Read access to my team's repo" | Read access to **every repo** visible to the user across **all orgs** |
| `repo:write` | "Push images to our CI repo" | Push access to **every repo** the user can write to, everywhere |
| `org:admin` | "Manage my team's org" | Full admin over **every org** the user belongs to, including `openshift` |

There is **no mechanism** to say: "This token can only read `myorg/myrepo` and nothing else."

**Code evidence:** In `auth/scopes.py`, scopes are defined as simple named constants (`READ_REPO`, `WRITE_REPO`, etc.) with no resource-binding capability. In `auth/permissions.py`, the `SCOPE_MAX_REPO_ROLES` mapping applies a blanket role cap across all repositories — the same cap applies to every repo the user has access to, with no per-repo filtering.

#### Problem 2: Only Organization Admins Can Manage Tokens

Token creation today requires navigating to **Organization Settings → Applications → Create OAuth Application → Generate Token**. This flow requires the `org:admin` role. Regular developers who simply need a pull token for their CI pipeline must:

1. Ask an org admin to create an OAuth Application
2. Wait for the admin to set it up
3. Then authorize it themselves

This does not scale for large organizations with hundreds of developers.

**Code evidence:** In `endpoints/api/organization_application_tokens.py`, all three endpoints (`GET`, `POST`, `DELETE`) require `ORG_ADMIN` scope, enforced via the `@require_scope(scopes.ORG_ADMIN)` decorator.

#### Problem 3: Token Creation Requires OAuth Application Expertise

Before a user can get a token, someone must first create an "OAuth Application" with a `client_id`, `client_secret`, `redirect_uri`, and `application_uri`. This is the standard OAuth 2.0 setup — appropriate for third-party integrations, but **hostile UX** for a developer who just wants an API key for their personal scripts.

**Code evidence:** In `data/database.py`, `OAuthAccessToken` has a **non-nullable** foreign key to `OAuthApplication`. There is no code path to create a token without an associated application.

### 1.3 The Architecture Root Cause

The fundamental issue is the **data model coupling**:

```
Token (OAuthAccessToken) 
  → belongs to → OAuthApplication (non-nullable FK) 
    → belongs to → Organization (QuayUserField FK to User where organization=True)
```

This chain enforces three constraints that this Phase 3 feature implementation must break:

1. Every token MUST be tied to an OAuth Application
2. Every OAuth Application MUST be tied to an Organization
3. Scopes are stored as flat strings (`"repo:read repo:write"`) with no resource binding

### 1.4 Security Impact

Tony's RICE score comment in the feature ticket describes the supply-chain risk:

> "Current quay.io personal tokens carry full org-admin permissions across ALL organizations a user belongs to, including the `openshift` org containing OCP release images."

The statement is not accurate. Not all tokens have org admin permissions. There is option to scope the tokens. But the scopes are also blanket across the namespaces. Also most tokens created a given org admin permissions due to CI/CD conviniece. A better more accurate way to state the current situations would be:
> "Quay's current scope model is namespace-wide: every scope applies blanket across all organizations and repositories a user has access to. There is no per-resource scoping. Combined with the practical tendency for users to select broad scopes like org:admin for CI/CD convenience, this creates a significant supply-chain risk — a single compromised token can grant an attacker full access across every organization the user belongs to."

**Note on terminology:** In the feature ticket the term "personal tokens" is used loosely to refer to existing OAuth Application tokens that individual users have authorized. True Personal Access Tokens (PATs) do not exist today — building them is the purpose of this feature.

A single compromised engineer laptop → full admin access across all orgs → potential deletion of critical OCP release images. Phase 3 directly addresses this by enabling tokens with minimal required permissions scoped to specific repositories.

### 1.5 What Phase 1 & Phase 2 Built (Foundation for Phase 3)

#### Phase 1 — Programmatic OAuth Token Provisioning (PROJQUAY-10436) - Already Implemented

**What it solved:** The chicken-and-egg problem — you need a token to create a token.

**What was built:**
- REST API endpoints to create/list/delete tokens programmatically
- Bootstrap token feature: auto-generates a token on first Quay startup
- Feature flag: `FEATURE_PROGRAMMATIC_BOOTSTRAP` (default: `false`)
- Shipped as Tech Preview in Quay 3.18
- Implementation PRs: [#6134](https://github.com/quay/quay/pull/6134) (design), [#6224](https://github.com/quay/quay/pull/6224) (code)

**Phase 3 relevance:** Phase 1 established the backend pattern for API-driven token creation. Phase 3 extends this pattern to user-level self-service.

**Deployment status:** The bootstrap feature is DISABLED on quay.io by two mechanisms: (1) `FEATURE_PROGRAMMATIC_BOOTSTRAP` defaults to `false`, (2) the `verify_not_prod` decorator blocks superuser methods when hostname contains "quay.io".

#### Phase 2 — OAuth Token Visibility & Management UI (PROJQUAY-9755) - Already Implemented

**What it solved:** Org admins had no visibility into tokens or ability to revoke individual tokens.

**What was built:**
- "API Access Tokens" tab in Organization Settings > Applications
- Token list: Name, Created By, Scopes, Expires, Last Used
- Granular revocation (per-token, not per-app)
- `display_name` column added to `OAuthAccessToken` via Alembic migration
- Implementation PRs: [#6461](https://github.com/quay/quay/pull/6461) (API), [#6488](https://github.com/quay/quay/pull/6488) (UI)

**Phase 2 API endpoints:**
```
GET    /api/v1/organization/{orgname}/applications/{client_id}/tokens
POST   /api/v1/organization/{orgname}/applications/{client_id}/tokens
DELETE /api/v1/organization/{orgname}/applications/{client_id}/tokens/{token_uuid}
```

**Key function `_can_mint_scope()`:** Validates that a user cannot create a token with more permissions than they actually hold. This pattern MUST be extended for Phase 3 to support per-repo validation.

**Phase 3 relevance:** Phase 3 reuses the token lifecycle API pattern and several UI components (token list table, revoke modal, token display modal).

**Deployment status:** Phase 2 API endpoints are NOT behind any feature flag — they are unconditionally imported in `endpoints/api/__init__.py`. They use `ORG_ADMIN` scope (not superuser), so they are likely available on quay.io.

---

## 2. Acceptance Criteria

> *Source: [PROJQUAY-6385](https://redhat.atlassian.net/browse/PROJQUAY-6385). Items marked [DESIGN NOTE] are engineering annotations.*

### 2.1 Token Creation

- Users can manage API tokens in their user account:
  - **Path 1 (OAuth Application flow):** They can create tokens in the context of an OAuth application following the existing "Application" concept from organizations (including application name, callback URL, etc.)
  - **Path 2 (Simplified flow):** They can create tokens alternatively using a simpler flow that only captures mandatory information: human-readable token name and optionally an expiry date
  - They can see a list of all tokens (both active and expired) as opposed to just a list of applications

> [DESIGN NOTE] Both paths produce user-level tokens with fine-grained scoping (confirmed with PM). The `token_kind = 'pat'` discriminator applies to ALL Phase 3 tokens regardless of creation path. The distinction is only in how the token is created (with or without an OAuth Application), not in what scope enforcement model it uses.

### 2.2 Organization Scoping

- Users can optionally limit API tokens to a single organization or a subset of organizations
  - The **default org scope** when creating a new token is **a single organization** selected by the user
  - Tokens spanning multiple organizations or all organizations require an explicit opt-in
  - When a user selects more than one org, or selects "All organizations I belong to," the UI displays a security notice:
    > *"This token will have access to resources across multiple organizations. For better security, consider creating a separate token per organization for each automation task."*

### 2.3 Repository Scoping

- Users can optionally limit API tokens to **specific repositories** within the selected organization(s)
  - The **default** is "all accessible repositories in the selected org(s)" → no change to existing behavior
  - Users can select individual repositories to **narrow** the token's scope
  - Repository-scoped tokens only grant access to the listed repos, even if the user has broader access within that org

### 2.4 Permission Scoping

- Users can optionally scope API tokens to a subset of permissions:
  - Organization management (CRUD access to robots, teams, membership and all org-level properties including org lifecycle) on existing orgs
  - Repository management (CRUD access to robots, teams and user and all repo-level properties including repo lifecycle) on existing repos
  - Organization creation
  - Repository creation
  - Repository read-access (view and pull)
  - Repository write-access (read/write to any accessible repositories)
  - User account administration
  - User account read-access

### 2.5 Escalation Prevention

- User-managed API tokens are always naturally limited by the permissions of the users owning the token:
  - The UI will **not allow** selecting more permissions than the user currently has
  - When user permissions are reduced over time, this affects the permission scope of the tokens owned by this user as well

> [DESIGN NOTE] Escalation prevention is enforced at the API level, not only the UI level. The `_can_mint_scope()` pattern from Phase 2 is extended for per-resource validation. See Section 4, Decision 8 for the full escalation prevention matrix.

### 2.6 Token Management UI

- Users will get a section in the UI to manage their API tokens, including:
  - Getting a list of existing tokens, with their name, scope, expiry date, and when last used
  - The ability to view the details of a particular token (and its OAuth application data if present)
  - The ability to refresh tokens in-place (shorthand for invalidating an existing token and recreating it with the same scope, name, settings, etc.)
  - The ability to delete tokens
  - The ability to change the expiry date of tokens (including expiring it right away)
- This user UI is supplemented by a new API

### 2.7 Phase 2 Relationship

- Phase 3 builds on Phase 2 UI components:
  - The token list table reuses the Phase 2 component (Name, Created By, Scopes, Expires, Last Used, Actions columns)
  - The "Create Token" wizard reuses the Phase 2 wizard, extended with the org-scope selector
  - **No changes** are made to Phase 2's "Org Settings → OAuth Applications → API Access Tokens tab"

### 2.8 Audit & Separation

- Tokens created in User Settings are labeled **"User Token"** in **audit logs** to distinguish them from org-admin tokens created via OAuth Applications (Phase 2)
- The "User Settings → API Tokens" page is a **new, standalone section**, clearly separated from Org Settings; there is no UI overlap or duplication with Phase 2

### 2.9 Defaults & Enforcement

- The simplified token creation flow (name + expiry only, no OAuth application setup required) defaults the org scope to a **single organization**
- Token permissions can **never exceed** the creating user's permissions in the selected organization(s), enforced at the API level, not only at the UI level

### 2.10 Resolved Open Questions

| Question | Resolution |
|---|---|
| Should we fold in organization application UI screens into the new user-based token management? | **No.** Phase 2 (PROJQUAY-9755) owns the organization-level token management UI. Phase 3 does not modify or replace that surface. |

```mermaid
graph TB
    subgraph "Phase 2 — Untouched"
        OrgSettings["Org Settings"] --> OAuthApps["OAuth Applications"]
        OAuthApps --> OrgTokens["API Access Tokens Tab"]
        OrgTokens --> BlanketScope["Blanket Scopes<br/>(existing behavior)"]
    end
    
    subgraph "Phase 3 — New"
        UserSettings["User Settings"] --> APITokens["API Tokens<br/>(new standalone section)"]
        APITokens --> Path1["Path 1: via OAuth App"]
        APITokens --> Path2["Path 2: Simplified Flow"]
        Path1 --> FineGrained["Fine-Grained Scopes<br/>per-org, per-repo, per-permission"]
        Path2 --> FineGrained
    end
    
    style BlanketScope fill:#fef3c7,stroke:#f59e0b
    style FineGrained fill:#dcfce7,stroke:#16a34a
    style OrgSettings fill:#2563eb,color:#fff
    style UserSettings fill:#16a34a,color:#fff
```

---

## 3. Solution Proposal

### 3.1 High-Level Architecture

Phase 3 introduces **Personal Access Tokens (PATs)** — tokens that belong directly to a user, support per-resource permission scoping, and do not require an OAuth Application intermediate.

**Key capabilities:**
1. Any authenticated user can create, list, and revoke their own tokens
2. Tokens can be scoped to specific organizations and/or repositories
3. Each resource binding carries a specific permission level (read, write, admin)
4. A simplified creation flow: name + expiry + scope selections (no OAuth App setup)
5. An optional OAuth Application flow for users who need third-party integrations
6. Multi-org support with security warnings for broad-scope tokens


### 3.1a Co-existence: PATs and Existing OAuth Application Tokens

Phase 3 PATs co-exist side by side with existing org-level OAuth Application tokens in the same `OAuthAccessToken` table:

| Aspect | Org OAuth App Tokens (Phase 2) | User PATs (Phase 3) |
|---|---|---|
| `token_kind` | `'oauth_app'` | `'pat'` |
| Created where | Org Settings → OAuth Applications | User Settings → API Tokens |
| Who creates | Org admins | Any authenticated user |
| Scope model | Blanket (existing, unchanged) | Fine-grained (per-org, per-repo) |
| `application` FK | NOT NULL (linked to org OAuth app) | NULL (Path 2) or set (Path 1) |
| Management UI | Phase 2 (unchanged) | Phase 3 (new) |
| Audit label | Org-admin token | "User Token" |

**Key rules:**
- Creating a PAT does NOT affect any existing OAuth app tokens — they are independent records
- Both are valid for API and Docker auth — the difference is how scopes are enforced
- Org admins have "two hats": they manage org tokens in Org Settings, and their own PATs in User Settings
- An org admin's PAT scoped to `repo:read` on one repo cannot push — even though the admin has full org access (the PAT's scope is the ceiling)

### 3.2 Data Model Changes

#### 3.2.1 Modified Table: `OAuthAccessToken`

Make the `application` foreign key **nullable** to allow tokens without an associated OAuth Application:

```sql
ALTER TABLE oauthaccesstoken 
  ALTER COLUMN application_id DROP NOT NULL;
```

Add a `token_kind` discriminator column to distinguish token types:

```sql
ALTER TABLE oauthaccesstoken
  ADD COLUMN token_kind VARCHAR(20) NOT NULL DEFAULT 'oauth_app';
  -- Values: 'oauth_app' (existing), 'pat' (Phase 3)
```

**Rationale for nullable `application` vs. hidden system app:**
- Option A (hidden system app per user): Creates artificial OAuthApplication records polluting the applications list, complicates queries, obscures the data model
- **Option B (nullable application) — RECOMMENDED**: Clean semantic separation, no phantom data, simpler queries for PATs (`WHERE application IS NULL`), aligns with how GitHub and GitLab model PATs

- **IMPORTANT:** [Refer to Section 7](#7-risk-of-making-application_id-nullable-and-mitigation-plan) for the risk of making `application_id` nullable in the `oauthaccesstoken` table and mitigation plan.


#### 3.2.2 New Table: `PersonalAccessTokenScope`

Stores per-resource permission bindings for PATs:

```sql
CREATE TABLE personalaccesstokenscope (
    id            SERIAL PRIMARY KEY,
    token_id      INTEGER NOT NULL REFERENCES oauthaccesstoken(id) ON DELETE CASCADE,
    scope         VARCHAR(80) NOT NULL,       -- e.g. 'repo:read', 'repo:write', 'org:admin'
    resource_type VARCHAR(20) NOT NULL,       -- 'repository' or 'organization'
    namespace     VARCHAR(255) NOT NULL,      -- org name (e.g. 'myorg')
    resource_name VARCHAR(255) NULL,          -- repo name (NULL if resource_type='organization')
    
    -- Composite index for enforcement lookups
    INDEX idx_pat_scope_lookup (token_id, resource_type, namespace, resource_name)
);
```

**Design rationale:** A separate table (Option B from the architecture gaps analysis) rather than encoding resources in the scope string because:
- Queryable: "Show me all tokens with access to repo X" is a simple SQL query
- Extensible: Adding new resource types (e.g., teams, builds) doesn't change the schema
- No string-length explosion: Tokens scoped to 100 repos don't create a 10KB scope string
- Cleaner enforcement: The permission layer can do a `WHERE token_id = ? AND namespace = ? AND resource_name = ?` check

#### 3.2.3 Modified Table: `OAuthApplication` (for Path 1 only)

To support user-level OAuth Applications (Path 1 from acceptance criteria):

```sql
ALTER TABLE oauthapplication
  ADD COLUMN owner_user_id INTEGER NULL REFERENCES "user"(id);
```

When `owner_user_id` is set, the application belongs to a user (not an org). When NULL, the existing `organization` FK is used (backward compatible).

#### 3.2.4 Entity Relationship Diagram

```mermaid
erDiagram
    User ||--o{ OAuthApplication : "owner_user_id (NEW)"
    User ||--o{ OAuthAccessToken : "authorized_user"
    OAuthApplication ||--o{ OAuthAccessToken : "application (nullable)"
    OAuthAccessToken ||--o{ PersonalAccessTokenScope : "token_id"
    Organization ||--o{ OAuthApplication : "organization (existing)"

    User {
        int id PK
        string username
        string email
        boolean organization
        boolean robot
    }

    OAuthApplication {
        string client_id PK
        string secure_client_secret
        string redirect_uri
        string application_uri
        int organization_id FK
        int owner_user_id FK "NEW - for user-level apps"
        string name
        string description
    }

    OAuthAccessToken {
        string uuid PK
        int application_id FK "NULLABLE (was NOT NULL)"
        int authorized_user_id FK
        string scope
        string display_name
        string token_kind "NEW: oauth_app or pat"
        string token_name
        string token_code
        string token_type
        datetime expires_at
        string data
        datetime last_accessed
        datetime created
    }

    PersonalAccessTokenScope {
        int id PK
        int token_id FK "NEW TABLE"
        string scope
        string resource_type
        string namespace
        string resource_name "NULL for org-level"
    }
```

**Key relationships:**
- `OAuthAccessToken.application_id` is now **nullable** — `NULL` for Path 2 PATs, set for Path 1 (user OAuth app) tokens and existing org-level tokens
- `OAuthAccessToken.token_kind` = `'pat'` for ALL Phase 3 tokens (both paths), `'oauth_app'` for existing org-level tokens
- `PersonalAccessTokenScope` stores fine-grained resource bindings for ALL tokens where `token_kind = 'pat'`
- `OAuthApplication.owner_user_id` is **new** — set for user-owned apps (Path 1), NULL for org-owned apps (existing)

**Scope enforcement rule:**
```
token_kind = 'oauth_app' → Blanket scope enforcement (existing, unchanged)
token_kind = 'pat'       → Fine-grained enforcement via PersonalAccessTokenScope
                            (applies to BOTH Path 1 and Path 2 tokens)
```

### 3.3 API Design

#### 3.3.1 New User Token Endpoints

All endpoints are under `/api/v1/user/tokens` and require authentication (no specific scope — any authenticated user).

```
GET    /api/v1/user/tokens
       → List all PATs for the authenticated user
       Query params: ?scope=repo:read (filter by scope)
                     ?namespace=myorg (filter by org)
                     ?page=1&per_page=50 (pagination)
       Response: [{uuid, display_name, scopes: [{scope, resource_type, namespace, resource_name}], 
                   expires_at, last_accessed, created, token_kind}]

POST   /api/v1/user/tokens
       → Create a new PAT
       Body: {
         "display_name": "My CI token",
         "expiration": 2592000,  // seconds (30 days)
         "scopes": [
           {"scope": "repo:read", "resource_type": "repository", "namespace": "myorg", "resource_name": "myrepo"},
           {"scope": "repo:write", "resource_type": "repository", "namespace": "myorg", "resource_name": "other-repo"},
           {"scope": "org:admin", "resource_type": "organization", "namespace": "teamorg"}
         ]
       }
       Response: {uuid, display_name, token: "<opaque-token-placeholder-starting-with-quay_pat_>", scopes: [...], expires_at, created}
       NOTE: Raw token is returned ONCE in this response, never again.

GET    /api/v1/user/tokens/{token_uuid}
       → Get details of a specific PAT
       Response: {uuid, display_name, scopes: [...], expires_at, last_accessed, created,
                  application: {name, client_id, ...} | null}

DELETE /api/v1/user/tokens/{token_uuid}
       → Revoke a specific PAT

PATCH  /api/v1/user/tokens/{token_uuid}
       → Update token metadata (display_name, expiration)
       → NOTE: Scopes CANNOT be modified after creation (security decision)

POST   /api/v1/user/tokens/{token_uuid}/refresh
       → Refresh/rotate a token (generates new token value, same scopes)
       Response: {uuid, token: "<opaque-token-placeholder-starting-with-quay_pat_>", expires_at}
```

#### 3.3.2 User OAuth Application Endpoints (Path 1)

If Path 1 (user-level OAuth apps) is in scope:

```
GET    /api/v1/user/applications
POST   /api/v1/user/applications
GET    /api/v1/user/applications/{client_id}
DELETE /api/v1/user/applications/{client_id}
GET    /api/v1/user/applications/{client_id}/tokens
POST   /api/v1/user/applications/{client_id}/tokens
DELETE /api/v1/user/applications/{client_id}/tokens/{token_uuid}
```

These mirror the existing organization-level endpoints but scoped to the user.

### 3.4 Token Prefix Convention

PATs should have a distinguishable prefix for security scanning and identification:

```
Existing OAuth tokens:  <no prefix> (plain 40-char random strings — confirmed in codebase)
Phase 3 PATs:           quay_pat_<40_char_random>
```

> **Note:** Quay tokens today have NO prefix at all. They are plain 40-character random strings generated via `random_string_generator()` in `data/model/oauth.py`. The `quay_pat_` prefix is a **new proposal** for Phase 3 tokens. Consider also adding `quay_oat_` for new org-level OAuth tokens in a follow-up (not for existing tokens — that would break integrations).

This enables:
- **GitHub Secret Scanning:** GitHub runs automated scanners on every public commit looking for patterns that match known token prefixes (e.g., `ghp_` for GitHub PATs, `glpat-` for GitLab PATs). Quay could register `quay_pat_` with the [GitHub Secret Scanning Partner Program](https://docs.github.com/en/code-security/secret-scanning/secret-scanning-partner-program), enabling automatic detection and notification when Quay PATs are leaked in public repositories.
- Quick visual identification in logs and debug output
- Programmatic distinction between token types without a database lookup

### 3.5 Feature Flag

```python
# config.py
FEATURE_USER_PAT = False  # default off for incremental rollout
```

All Phase 3 endpoints and UI components should be gated behind this flag using the `@show_if` decorator:

```python
@show_if(features.USER_PAT)
class UserPersonalAccessTokens(ApiResource):
    ...
```

### 3.6 Scope Enforcement (Runtime)

This is the most critical and complex part of the implementation.

#### 3.6.1 Current Enforcement Flow

```
API Request with Bearer token
  → auth/auth_context.py: validate_oauth_token() identifies the user + scopes
  → auth/permissions.py: QuayDeferredPermissionUser loads permissions lazily
    → _populate_repository_provides(): loads repo-level permissions from DB
      → _repo_role_for_scopes(): caps the role based on token's scope set
        → SCOPE_MAX_REPO_ROLES[scope] → maximum role string
  → Flask-Principal: permission.can() checks loaded needs against requirements
```

**The injection point:** `_repo_role_for_scopes()` in `auth/permissions.py` (called around line 258 inside `_populate_repository_provides()`). Currently, this method applies a blanket cap:

```python
# Pseudocode of current behavior:
def _repo_role_for_scopes(self, role):
    # For EVERY repo, cap the role based on the token's scope set
    max_role = max(SCOPE_MAX_REPO_ROLES[s] for s in self.token_scopes)
    return min(role, max_role)  # cap at whatever the scope allows
```

#### 3.6.2 Phase 3 Enforcement Changes

For PATs (`token_kind = 'pat'`), add a **resource binding check** before the blanket scope cap:

```python
# Pseudocode of Phase 3 behavior:
def _repo_role_for_scopes(self, namespace, repo_name, role):
    if self.token_kind == 'pat':
        # Check if this specific (namespace, repo_name) is in the token's resource bindings
        binding = self._get_pat_scope_binding(namespace, repo_name)
        if binding is None:
            # Also check for org-level wildcard: if the token has an org-level scope
            # for this namespace, allow it
            org_binding = self._get_pat_org_binding(namespace)
            if org_binding is None:
                return None  # No access — this repo is not in the token's scope
            binding = org_binding
        # Cap at the binding's scope level
        max_role = SCOPE_MAX_REPO_ROLES[binding.scope]
        return min(role, max_role) if max_role else None
    else:
        # Existing behavior for oauth_app tokens — blanket cap
        max_role = max(SCOPE_MAX_REPO_ROLES[s] for s in self.token_scopes)
        return min(role, max_role)
```

**Performance consideration:** PAT scope lookups must be efficient. Options:
- Eager load all `PersonalAccessTokenScope` records when the token is validated (good for tokens with few scopes)
- Cache scope bindings in a dictionary keyed by `(namespace, resource_name)` for O(1) lookup
- For tokens with many scopes (100+), consider a Redis cache

#### 3.6.3 Docker Auth Flow Enforcement

Docker registry v2 authentication goes through a separate endpoint that issues short-lived JWT bearer tokens. This flow:

```
docker pull myorg/myrepo:latest
  → GET /v2/ → 401 with WWW-Authenticate header
  → GET /v2/auth?scope=repository:myorg/myrepo:pull → Bearer JWT
  → GET /v2/myorg/myrepo/manifests/latest with Bearer JWT
```

The `/v2/auth` endpoint validates the OAuth token and checks if it has the required scope for the requested repository. **Phase 3 will add per-resource filtering here too** — when the OAuth token is a PAT, verify that `(myorg, myrepo)` is in the token's `PersonalAccessTokenScope` bindings before issuing the JWT bearer token.

### 3.7 Alembic Migration

```python
"""add_personal_access_token_support

Revision ID: <auto_generated>
Revises: b30800b1d271
"""

import sqlalchemy as sa

def upgrade(op, tables, tester):
    # 1. Make application nullable on oauthaccesstoken
    with op.batch_alter_table("oauthaccesstoken") as batch_op:
        batch_op.alter_column("application_id", nullable=True)
    
    # 2. Add token_kind discriminator
    op.add_column(
        "oauthaccesstoken",
        sa.Column("token_kind", sa.String(length=20), nullable=False, server_default="oauth_app")
    )
    
    # 3. Create PersonalAccessTokenScope table
    op.create_table(
        "personalaccesstokenscope",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("token_id", sa.Integer, sa.ForeignKey("oauthaccesstoken.id", ondelete="CASCADE"), nullable=False),
        sa.Column("scope", sa.String(length=80), nullable=False),
        sa.Column("resource_type", sa.String(length=20), nullable=False),
        sa.Column("namespace", sa.String(length=255), nullable=False),
        sa.Column("resource_name", sa.String(length=255), nullable=True),
    )
    op.create_index(
        "idx_pat_scope_lookup",
        "personalaccesstokenscope",
        ["token_id", "resource_type", "namespace", "resource_name"]
    )
    
    # 4. Add owner_user_id to oauthapplication (for Path 1)
    op.add_column(
        "oauthapplication",
        sa.Column("owner_user_id", sa.Integer, sa.ForeignKey("user.id"), nullable=True)
    )
    
    # Populate test data
    tester.populate_column("oauthaccesstoken", "token_kind", tester.TestDataType.String)

def downgrade(op, tables, tester):
    op.drop_column("oauthapplication", "owner_user_id")
    op.drop_table("personalaccesstokenscope")
    op.drop_column("oauthaccesstoken", "token_kind")
    with op.batch_alter_table("oauthaccesstoken") as batch_op:
        batch_op.alter_column("application_id", nullable=False)
```

---

## 4. Key Design Decisions

### Decision 1: Nullable `application` FK vs. Hidden System App

| Criteria | Nullable FK (Recommended) | Hidden System App |
|---|---|---|
| Data model clarity | Clean — PATs have no application | Artificial phantom records |
| Query simplicity | `WHERE application IS NULL AND token_kind='pat'` | Must filter out system apps from app listings |
| Migration risk | Medium — must audit all code assuming non-null application | Low — no schema change |
| GitHub/GitLab precedent | Both use separate PAT models | No major platform uses this pattern |

**Decision: Use nullable `application` FK with `token_kind` discriminator.**

All code paths that currently assume `OAuthAccessToken.application` is non-null must be audited and updated. A `token_kind` field makes it easy to branch logic.

### Decision 2: Per-Resource Scope Storage

| Criteria | Separate Table (Recommended) | Encoded in Scope String |
|---|---|---|
| Queryability | "Find all tokens accessing repo X" | Requires LIKE '%myorg/myrepo%' |
| Scalability | Efficient joins and indexes | String length grows with # of repos |
| Extensibility | Add new resource types via new rows | Scope string format becomes complex |
| Enforcement performance | O(1) hash lookup after eager load | Parsing on every request |

**Decision: Use `PersonalAccessTokenScope` table.**

### Decision 3: Scope Immutability After Creation

**Decision: Token scopes CANNOT be modified after creation.** Users must create a new token if they need different scopes.

**Rationale:**
- Security: Prevents privilege escalation on an existing, possibly compromised token
- Audit trail: The scopes at creation time are the scopes for the token's lifetime
- Simplicity: Avoids complex "scope diff" logic and race conditions
- GitHub precedent: GitHub fine-grained PATs are also immutable after creation

### Decision 4: Token Prefix for Detection

**Decision: Use `quay_pat_` prefix for all Phase 3 PATs.**

This enables:
- Secret scanning tools (GitHub, GitLab, Trufflehog) to detect leaked Quay PATs
- Quick visual identification in logs and debug output
- Programmatic distinction between token types without a database lookup

### Decision 5: Org-Level vs. Repo-Level Scoping

**Decision: Support BOTH org-level and repo-level scoping.**

- **Org-level scope:** "repo:read on ALL repos in orgA (current and future)" — stored as `resource_type='organization'`, `resource_name=NULL`
- **Repo-level scope:** "repo:read on orgA/myrepo ONLY" — stored as `resource_type='repository'`, `resource_name='myrepo'`

When both exist for the same namespace, the more specific (repo-level) takes precedence.

### Decision 5a: Scope Combination Examples

The following examples show how `PersonalAccessTokenScope` rows map to different access patterns:

**Scenario A: Specific repos only**
Token has `repo:read` on `myorg1/myrepo1`, `myorg1/myrepo2`, and `myorg2/myrepo3`:

| token_id | scope | resource_type | namespace | resource_name |
|---|---|---|---|---|
| token_1 | repo:read | repository | myorg1 | myrepo1 |
| token_1 | repo:read | repository | myorg1 | myrepo2 |
| token_1 | repo:read | repository | myorg2 | myrepo3 |

*Result:* Can pull from exactly these 3 repos. Nothing else.

**Scenario B: All repos in selected orgs**
Token has `repo:write` on ALL repos across two orgs:

| token_id | scope | resource_type | namespace | resource_name |
|---|---|---|---|---|
| token_2 | repo:write | organization | myorg1 | NULL |
| token_2 | repo:write | organization | myorg2 | NULL |

*Result:* `resource_type=organization` + `resource_name=NULL` means "all repos in this org (current AND future)." The user selects each org explicitly — there is no "all orgs" wildcard.

**Scenario C: All repos in one org only**

| token_id | scope | resource_type | namespace | resource_name |
|---|---|---|---|---|
| token_3 | repo:write | organization | myorg1 | NULL |

*Result:* Can push to any repo in myorg1 (current and future). Cannot access myorg2.

**Scenario D: Org-wide read + admin on one specific repo**

| token_id | scope | resource_type | namespace | resource_name |
|---|---|---|---|---|
| token_4 | repo:read | organization | myorg1 | NULL |
| token_4 | repo:admin | repository | myorg1 | critical-repo |

*Result:* Read all repos in myorg1. Admin only on `critical-repo`. **Resolution rule:** Repo-level binding overrides org-level binding for that specific repo (more-specific wins).

**Scenario E: Full org management**

| token_id | scope | resource_type | namespace | resource_name |
|---|---|---|---|---|
| token_5 | org:admin | organization | myorg1 | NULL |

*Result:* Full admin on myorg1 (create/delete repos, manage teams, manage robots). No access to myorg2.

### Decision 6: Multi-Org Token Support

**Decision: A single PAT can cover resources in multiple organizations.**

**UI mitigation:** When a user creates a token spanning 2+ orgs, display a warning:

> ⚠️ This token will have access to resources in multiple organizations: orgA, orgB. A single compromised token would affect all listed organizations. Consider creating separate tokens per organization for better security isolation.

### Decision 7: Feature Flag Strategy

**Decision: Gate behind `FEATURE_USER_PAT` (default: `false`).**

This allows:
- Incremental rollout on quay.io
- Self-hosted customers to opt-in when ready
- Easy rollback if issues are discovered

### Decision 8: Escalation Prevention

**Decision: Users cannot create a PAT with permissions exceeding their own access level on any targeted resource.**

Extend the Phase 2 `_can_mint_scope()` pattern:
- For each scope + resource binding, verify the user's actual permission on that resource
- If the user has `repo:read` on `myorg/myrepo`, they cannot create a PAT with `repo:admin` on `myorg/myrepo`
- If the user has no access to `otherorg/secretrepo`, they cannot create a PAT targeting it

**Scope vs. Permission — key distinction:**
```
SCOPE      = What the TOKEN is allowed to do (ceiling set at token creation)
PERMISSION = What the USER is actually allowed to do (from org/team/repo membership)

Effective access = min(scope, permission)
```
Scopes can only *reduce* access, never *increase* it.

**Escalation prevention matrix:**

| User's actual permission | Requested PAT scope | Result |
|---|---|---|
| `repo:read` on myorg/myrepo | `repo:read` | Allowed |
| `repo:read` on myorg/myrepo | `repo:write` | Rejected (escalation) |
| `repo:read` on myorg/myrepo | `repo:admin` | Rejected (escalation) |
| `repo:write` on myorg/myrepo | `repo:read` | Allowed (downgrade) |
| `repo:write` on myorg/myrepo | `repo:write` | Allowed (exact match) |
| `repo:write` on myorg/myrepo | `repo:admin` | Rejected (escalation) |
| `repo:admin` on myorg/myrepo | `repo:read` | Allowed (downgrade) |
| `repo:admin` on myorg/myrepo | `repo:admin` | Allowed (exact match) |
| No access to otherorg/secretrepo | any scope | Rejected (no access) |
| Org member (not admin) of myorg | `org:admin` on myorg | Rejected (not org admin) |
| Org admin of myorg | `org:admin` on myorg | Allowed |
| Org admin of myorg | `org:admin` on otherorg | Rejected (not admin there) |

**Validation rules at PAT creation:**
1. For each scope+resource binding, check if user has ≥ that permission on that specific resource
2. Use the implied scope hierarchy: `repo:admin` ≥ `repo:write` ≥ `repo:read`
3. If *any* binding fails validation, reject the *entire* PAT creation (not partial)
4. Return a clear error: `"You do not have repo:write permission on myorg/myrepo"`

### Decision 9: Maximum Token Lifetime

**Decision: Configurable maximum, defaulting to 365 days.**

```python
# config.py
PAT_MAX_EXPIRATION_SECONDS = 31536000  # 365 days, configurable
PAT_DEFAULT_EXPIRATION_SECONDS = 2592000  # 30 days
```

The current 10-year default for OAuth tokens is excessive for fine-grained PATs. Organizations should be able to enforce shorter lifetimes.

### Decision 10: Backward Compatibility

**Decision: Existing OAuth Application tokens are completely unaffected.**

- No migration of existing tokens to the PAT model
- The `token_kind` column defaults to `'oauth_app'` for all existing records
- Existing API endpoints (`/v1/organization/{orgname}/applications/{client_id}/tokens`) continue to work unchanged
- Phase 2 UI in Organization Settings is unmodified

---

## 5. User Journeys

### Journey 1: Developer Creates a Read-Only Pull Token

**Persona:** A developer who needs to pull images from `myteam/app-frontend` in their CI pipeline.

**Steps:**
1. User navigates to **User Settings → Personal Access Tokens**
2. Clicks **"Create New Token"**
3. Enters token name: `"CI pipeline pull token"`
4. Sets expiration: `90 days`
5. In the scope selection:
   - Selects organization: `myteam`
   - Selects repository: `app-frontend`
   - Selects permission: `Read (pull only)`
6. Reviews the summary showing the blast radius:
   > This token can: Pull images from `myteam/app-frontend`. No other access.
7. Clicks **"Generate Token"**
8. Copies the displayed token (shown once)
9. Pastes it into their CI pipeline's `QUAY_TOKEN` secret

**Backend flow:**
```
POST /api/v1/user/tokens
{
  "display_name": "CI pipeline pull token",
  "expiration": 7776000,
  "scopes": [
    {"scope": "repo:read", "resource_type": "repository", "namespace": "myteam", "resource_name": "app-frontend"}
  ]
}
```

### Journey 2: DevOps Engineer Creates a Multi-Repo Push Token

**Persona:** A DevOps engineer who needs to push images to 3 repos across 2 orgs.

**Steps:**
1. Navigates to **User Settings → Personal Access Tokens**
2. Clicks **"Create New Token"**
3. Enters name: `"Build pipeline push token"`
4. Sets expiration: `30 days`
5. Adds scopes:
   - `myteam/app-frontend` → Read/Write
   - `myteam/app-backend` → Read/Write
   - `shared-images/base-images` → Read only
6. UI shows multi-org warning:
   > ⚠️ This token spans 2 organizations (myteam, shared-images).
7. Reviews and generates

**Backend flow:**
```
POST /api/v1/user/tokens
{
  "display_name": "Build pipeline push token",
  "expiration": 2592000,
  "scopes": [
    {"scope": "repo:write", "resource_type": "repository", "namespace": "myteam", "resource_name": "app-frontend"},
    {"scope": "repo:write", "resource_type": "repository", "namespace": "myteam", "resource_name": "app-backend"},
    {"scope": "repo:read", "resource_type": "repository", "namespace": "shared-images", "resource_name": "base-images"}
  ]
}
```

### Journey 3: Team Lead Creates an Org-Wide Admin Token

**Persona:** A team lead who needs full admin access to their org for scripting.

**Steps:**
1. Creates a PAT with:
   - Name: `"Org management scripts"`
   - Expiration: `7 days` (short for sensitive scope)
   - Scope: `myteam` (organization) → `org:admin`
2. UI shows elevated-privilege warning:
   > 🔴 This token has ADMIN access to the entire `myteam` organization, including all current and future repositories. Use the shortest practical expiration.
3. Reviews and generates

**Backend flow:**
```
POST /api/v1/user/tokens
{
  "display_name": "Org management scripts",
  "expiration": 604800,
  "scopes": [
    {"scope": "org:admin", "resource_type": "organization", "namespace": "myteam"}
  ]
}
```

### Journey 4: User Views and Revokes a Token

**Steps:**
1. Navigates to **User Settings → Personal Access Tokens**
2. Sees a list of their tokens with columns:
   - Name | Scopes Summary | Expires | Last Used | Created
3. Clicks on a token to see details:
   - All resource bindings with permissions
   - OAuth Application data (if created via Path 1)
   - Last accessed timestamp
4. Clicks **"Revoke"** on an expired or compromised token
5. Confirms revocation in a modal

### Journey 5: Docker Pull With a Scoped PAT

**Persona:** A developer using a PAT scoped to `myteam/app-frontend` (repo:read).

**Success case:**
```bash
$ docker login quay.io -u myuser -p quay_pat_abc123...
Login Succeeded

$ docker pull quay.io/myteam/app-frontend:latest
latest: Pulling from myteam/app-frontend
...
Status: Downloaded newer image
```

**Failure case (out of scope):**
```bash
$ docker pull quay.io/myteam/app-backend:latest
Error response from daemon: unauthorized: access to the requested resource 
is not authorized. Token does not have access to repository myteam/app-backend.
```

**Implementation note:** The v2 auth endpoint must check `PersonalAccessTokenScope` bindings when the OAuth token is a PAT, and return a clear error message indicating the specific resource is not in the token's scope.

### Journey 6: Token Refresh

**Steps:**
1. User navigates to token details
2. Clicks **"Refresh Token"**
3. UI warns: _"This will generate a new token value. The old value will stop working immediately. Are you sure?"_
4. User confirms
5. New token value is displayed (shown once)
6. Old token value is invalidated

**Backend flow:**
```
POST /api/v1/user/tokens/{token_uuid}/refresh
Response: {uuid: same, token: "<opaque-token-placeholder-starting-with-quay_pat_>", expires_at: <extended>}

```

---

## 6. Limitations and Clarifications Required

### 6.1 Clarifications Needed from Product Owner

#### Scope Model (Critical — Blocks Design)

| # | Question | Why It Matters | Recommendation |
|---|---|---|---|
| 1 | **Wildcard org scoping?** Should users be able to say "all current AND future repos in this org"? | Determines if `resource_type='organization'` means "all repos now" or "all repos now and forever" | Recommend: org-level scope covers current AND future repos (otherwise it's unusable for dynamic orgs) |
| 2 | **Cross-org tokens?** Can one PAT cover repos in multiple organizations? | The ticket says "multi-org support" but the security implications are significant | Recommend: Allow but with prominent warning in UI |
| 3 | **Scope granularity beyond read/write/admin?** Should there be `create`, `delete`, `manage-tags`, `manage-mirrors`? | Affects the `PersonalAccessTokenScope.scope` enum and enforcement complexity | Recommend: Start with existing scopes (repo:read, repo:write, repo:admin, org:admin, user:read, user:admin). Add finer granularity in a follow-up |

#### Permissions & Security

| # | Question | Why It Matters | Recommendation |
|---|---|---|---|
| 4 | **Robot accounts?** Can robot accounts create PATs? | Robots are used heavily for automation; PATs could replace robot tokens | Recommend: Phase 1 = human users only. Evaluate robot PATs as follow-up |
| 5 | **Org admin visibility?** Should org admins see/revoke PATs that access their org? | Tension between user autonomy and org governance | Recommend: Org admins can VIEW (not revoke) PATs accessing their org. Org owner can revoke (with notification to user) |
| 6 | **Notification on revocation?** Should users be notified when their PAT is revoked by an org admin? | Impacts audit trail and user experience | Recommend: Yes, email + in-app notification |

#### Lifecycle

| # | Question | Why It Matters | Recommendation |
|---|---|---|---|
| 7 | **Max tokens per user?** Is there a limit? | Prevents abuse and simplifies billing/resource planning | Recommend: Configurable limit, default 100 |
| 8 | **Token refresh semantics?** Does refresh extend expiry or just rotate the secret? | The ticket mentions "token refresh" but doesn't specify behavior | Recommend: Refresh rotates the secret AND resets expiry to the original duration |
| 9 | **Expiration notification?** Should users be warned before token expiry? | Improves UX for CI/CD tokens that would silently break | Recommend: Email notification 7 days before expiry |

#### UX & Deployment

| # | Question | Why It Matters | Recommendation |
|---|---|---|---|
| 10 | **UI location?** User Settings page? New top-level nav? Both? | Impacts frontend routing and navigation | Recommend: User Settings → "Personal Access Tokens" tab |
| 11 | **Path 1 phasing?** Can we ship Path 2 (PATs) first and add Path 1 (user OAuth apps) as follow-up? | Path 1 significantly increases scope; Path 2 delivers 90% of the security value | Recommend: Phase Path 1 as separate work item |
| 12 | **Docker error messages?** What should users see when a scoped PAT denies access to a repo? | Affects user experience and debugging | Recommend: Clear error stating the specific repo is not in the token's scope |

### 6.2 Known Limitations

#### Limitation 1: Existing Token Migration

There is **no automated migration path** from existing OAuth Application tokens to PATs. Users with existing tokens must:
1. Create new PATs with the desired scopes
2. Update their scripts/CI to use the new PATs
3. Revoke old OAuth tokens manually

**Rationale:** Automatic migration is unsafe — the system cannot infer what fine-grained scopes a blanket-scoped token _should_ have.

#### Limitation 2: Scope Cannot Be Modified After Creation

Once a PAT is created, its scopes are immutable. To change scopes, the user must create a new token and revoke the old one.

**Rationale:** Immutability prevents privilege escalation on potentially compromised tokens and simplifies the audit trail.

#### Limitation 3: No Cross-Registry Scoping

PATs only apply to the Quay instance they were created on. A PAT created on `quay.io` cannot be used on a self-hosted Quay instance or vice versa.

#### Limitation 4: Backward Compatibility with `docker login`

`docker login` uses a single token for all subsequent `docker pull`/`push` operations to a registry hostname. If a user has a PAT scoped to `orgA/repo1` but tries to `docker pull orgA/repo2`, they will get an auth error — there is no "partial login" concept in Docker.

**Mitigation:** Clear error messages and documentation explaining this behavior.

#### Limitation 5: Scope Enforcement Performance

For tokens with many resource bindings (100+), the eager-load-and-hash approach may add latency to the first API call per session. If this becomes an issue, consider:
- Caching PAT scope lookups in Redis with TTL
- Setting a per-token scope binding limit (e.g., 200 repos)


### 6.6 Scope Enforcement Performance Detail

#### Strategy 1: Eager Load + In-Memory Dictionary (Recommended for v1)

When a PAT is first validated in a request, load all its scope bindings into a Python dictionary for the duration of the request:

```python
# On first access in the request:
scope_bindings = {}
for row in PersonalAccessTokenScope.select().where(
    PersonalAccessTokenScope.token_id == token.id
):
    key = (row.namespace, row.resource_name)  # ('myorg', 'myrepo') or ('myorg', None)
    scope_bindings[key] = row.scope

# On each permission check during the request:
def has_scope_for_resource(namespace, repo_name):
    # Check specific repo binding first (more-specific wins)
    if (namespace, repo_name) in scope_bindings:
        return scope_bindings[(namespace, repo_name)]
    # Fall back to org-level binding
    if (namespace, None) in scope_bindings:
        return scope_bindings[(namespace, None)]
    return None  # No access
```

**Why this works:** Most PATs will have 1–20 scope bindings. Loading 20 rows into a dictionary on the first request is negligible (~0.1ms). All subsequent permission checks in the same request are O(1) dictionary lookups.

#### Strategy 2: Redis Cache (Follow-up for High-Traffic Tokens)

For CI pipelines making hundreds of requests/second with the same PAT:

```python
cache_key = f"pat_scopes:{token.id}"
cached = redis.get(cache_key)
if cached:
    scope_bindings = deserialize(cached)
else:
    scope_bindings = load_from_db(token.id)
    redis.setex(cache_key, 300, serialize(scope_bindings))  # 5-min TTL
```

**Cache invalidation:** When a PAT is revoked, delete its Redis key. Since PAT scopes are immutable after creation (Decision 3), there is no risk of stale scope data — only revocation needs to invalidate the cache.

| Strategy | DB queries per request | Lookup time | Infrastructure |
|---|---|---|---|
| Naive (query per check) | 10+ per request | ~1ms each | None |
| **Strategy 1 (eager load)** | **1 per request** | **O(1) after load** | **None** |
| Strategy 2 (Redis) | 0 (cache hit) | O(1) | Redis required |

**Recommendation:** Start with Strategy 1. Add Redis only if production monitoring shows latency issues with high-traffic tokens.

### 6.3 Inadequacies in the Current Ticket Description

| # | Issue | Detail |
|---|---|---|
| 1 | **"Simplified flow: name + expiry only"** is incomplete | The simplified flow MUST include scope/resource selection. Without it, the token is just another blanket token — defeating the purpose |
| 2 | **Scope model is unspecified** | How per-repo permissions are encoded and enforced is the single biggest design decision and it's not in the ticket |
| 3 | **"Reuses Phase 2 UI components"** is overestimated | Phase 2 UI is org-admin-centric. Phase 3 needs new page structure, new data flows, and new components for repo/org selection |
| 4 | **No mention of authorization enforcement** | The ticket describes creation UX but not runtime enforcement — every API call and Docker operation must check resource bindings |
| 5 | **Dependency chain is incomplete** | Phase 2 is not listed as a dependency, but Phase 3 reuses Phase 2 API patterns and UI components |
| 6 | **PM terminology is imprecise** | "Personal tokens" in comments refers to existing OAuth tokens, not true PATs. This should be clarified to avoid confusion |

### 6.4 Edge Cases to Consider

| Edge Case | Expected Behavior |
|---|---|
| User loses access to a repo after PAT creation | Token's scope binding still exists, but enforcement denies access because the underlying user permission check fails. The `_populate_repository_provides()` method won't load a need for a repo the user can't access. |
| Org is deleted while PAT has scopes targeting it | PAT scope bindings become orphaned. Enforcement naturally denies access because the namespace doesn't resolve. Consider cleanup job. |
| Repo is renamed | PAT scope bindings reference the old name. Access will fail. The user must create a new PAT. Consider supporting repo rename event hooks. |
| User creates PAT with 0 scopes | Reject at API level. A PAT with no scopes is useless and confusing. |
| User creates PAT for a repo they own but then transfer ownership | Same as "user loses access" — enforcement checks underlying permissions. |
| Token expiration at exactly midnight UTC | Use `expires_at > utcnow()` for comparison, not `>=`. Consistent with existing token expiration logic. |
| Concurrent refresh requests | Use optimistic locking (check-and-set) or database-level row locking to prevent race conditions during token rotation. |

### 6.5 Files Requiring Changes

| File | Change Type | Description |
|---|---|---|
| `data/database.py` | Modify | Make `OAuthAccessToken.application` nullable, add `token_kind`, add `PersonalAccessTokenScope` model, add `OAuthApplication.owner_user_id` |
| `data/model/oauth.py` | Modify + Add | Add PAT-specific CRUD functions, extend validation functions |
| `data/migrations/versions/` | Add | New Alembic migration for schema changes |
| `auth/scopes.py` | Modify | Add PAT-aware scope validation functions |
| `auth/permissions.py` | Modify | Add per-resource scope filtering in `_repo_role_for_scopes()` and `_populate_repository_provides()` |
| `auth/auth_context.py` | Modify | Handle PAT token type in `get_validated_oauth_token()` |
| `endpoints/api/user_tokens.py` | Add (new file) | All PAT API endpoints |
| `endpoints/api/__init__.py` | Modify | Import and register new endpoints (with feature flag gating) |
| `config.py` | Modify | Add `FEATURE_USER_PAT`, `PAT_MAX_EXPIRATION_SECONDS`, `PAT_DEFAULT_EXPIRATION_SECONDS` |
| `web/src/resources/UserTokenResource.ts` | Add (new file) | API client for PAT endpoints |
| `web/src/resources/UserTokenTypes.ts` | Add (new file) | TypeScript type definitions |
| `web/src/hooks/UseUserTokens.ts` | Add (new file) | React hooks for PAT operations |
| `web/src/routes/UserSettings/PersonalAccessTokens/` | Add (new dir) | PAT list page, create wizard, detail view |
| `web/src/components/modals/` | Modify | Extend or create modals for PAT generation/revocation |
| `endpoints/v2/` | Modify | Add PAT scope filtering to Docker v2 auth endpoint |

---

## 7. Risk of making application_id nullable and mitigation plan

These location in the codebase would break as they accesses `.application` on a token and assumes it's non-null:

#### File 1: auth/context_entity.py (lines 169-170)
```
python
entity_reference.application.client_id   # ← NullPointerError for PATs
entity_reference.application.name        # ← NullPointerError for PATs
```
**Used in:** Audit logging — writes the application's `client_id` and name into audit events.

#### File 2: auth/oauth.py (lines 146, 151, 153-154, 171, 176)
```
python
validated.application.organization.username  # ← NullPointerError
validated.application.name                   # ← NullPointerError
validated.application.client_id              # ← NullPointerError
```
**Used in:** OAuth token validation and org namespace resolution.

#### File 3: data/model/oauth.py (~7 locations)
```
python
Line 481: OAuthAccessToken.application == application  # query filter — OK if nullable
Line 717: found.application_id != canonical_application.id  # comparison — needs guard
Line 978: OAuthAccessToken.create(application=application)  # creation — already your code
```
Query filters with `WHERE application = X` are safe even with nullable FK. The risk is in attribute access (.application.something).

#### File 4: data/database.py — Meta indexes
```
python
Meta.indexes = ((("application", "last_accessed"), False),)
```
This composite index works fine with NULL values — PostgreSQL indexes include NULLs by default.

### Mitigation strategy:

```
python
# Add a helper property on OAuthAccessToken:
@property
def is_pat(self):
    return self.token_kind == 'pat'

# Then guard every .application access:
if not token.is_pat:
    client_id = token.application.client_id
else:
    client_id = "pat"  # or token.uuid for audit identification
```

**Total risk assessment:** ~10 code locations across 3 files need null-guards. The risk is manageable — all are in well-defined auth/audit paths. The alternative (hidden system app) avoids these changes but introduces worse conceptual complexity.

---

## Appendix A: Reference Material

### Key PRs to Study

| PR | Phase | Description |
|---|---|---|
| [#6134](https://github.com/quay/quay/pull/6134) | Phase 1 | Design document for programmatic token provisioning |
| [#6224](https://github.com/quay/quay/pull/6224) | Phase 1 | Implementation |
| [#6461](https://github.com/quay/quay/pull/6461) | Phase 2 | API + data model (token lifecycle endpoints, `display_name` migration) |
| [#6488](https://github.com/quay/quay/pull/6488) | Phase 2 | UI (token list, generate wizard, revoke action, Playwright e2e tests) |

### Recommended Reading Order

1. `auth/scopes.py` — scope definitions and hierarchy
2. `data/database.py` — ORM models (`OAuthApplication`, `OAuthAccessToken`)
3. `data/model/oauth.py` — business logic layer
4. `auth/permissions.py` — enforcement engine (focus on `_repo_role_for_scopes`, `_populate_repository_provides`)
5. `endpoints/api/organization_application_tokens.py` — Phase 2 API (your template)
6. `auth/auth_context.py` — authentication flow
7. PR #6461 diff — Phase 2 implementation details
8. PR #6488 diff — Phase 2 UI patterns

### External References

- [GitHub Fine-Grained PATs Documentation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)
- [GitLab Personal Access Tokens](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)
- [OAuth 2.0 RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)

---
