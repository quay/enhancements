# Kubernetes ServiceAccount OIDC Authentication

**Status:** Proposed
**JIRA:** PROJQUAY-0000
**Authors:** Brady Pratt

## Summary

Enable Kubernetes ServiceAccounts to authenticate to Quay using OIDC federation. This allows Kubernetes operators (like quay-operator) to authenticate to Quay using their pod's projected service account token instead of static credentials.

## Motivation

Kubernetes workloads currently require static robot account credentials to interact with Quay. This creates operational overhead for credential rotation and introduces security risks from long-lived secrets. Kubernetes ServiceAccount tokens are short-lived, automatically rotated, and bound to specific audiences, making them a more secure authentication mechanism.

### Goals

- Allow configured Kubernetes ServiceAccounts to authenticate using projected SA tokens
- Map authenticated SAs to robot accounts in a dedicated system organization
- Grant superuser permissions to configured SA subjects
- Validate token audience to prevent replay attacks

### Non-Goals

- UI for managing Kubernetes SA authentication
- OIDC browser-based login flows (this is API/bearer token only)
- Authentication for arbitrary Kubernetes SAs (only explicitly configured subjects)

## Proposal

### Architecture

```
                                    ┌──────────────────────┐
                                    │  Kubernetes Cluster  │
                                    │                      │
┌─────────────────┐                 │  ┌───────────────┐   │
│     Quay        │ ◄──────────────────│  Operator Pod │   │
│                 │   Bearer Token  │  │  (with SA)    │   │
│ ┌─────────────┐ │                 │  └───────────────┘   │
│ │auth/oauth.py│ │                 │                      │
│ │             │ │                 │  ┌───────────────┐   │
│ │ validate_   │ │  JWKS Fetch     │  │  K8s API      │   │
│ │ kubernetes_ │──────────────────────│  Server       │   │
│ │ sa_token()  │ │                 │  │  (OIDC/JWKS)  │   │
│ └─────────────┘ │                 │  └───────────────┘   │
└─────────────────┘                 └──────────────────────┘
```

### Authentication Flow

1. **Token Presentation:** Operator pod sends bearer token to Quay API
2. **Issuer Check:** Quay extracts issuer from token, matches against configured OIDC server
3. **JWKS Validation:** Token signature validated against Kubernetes OIDC JWKS endpoint
4. **Audience Validation:** Token `aud` claim validated against expected audience (default: "quay")
5. **Subject Authorization:** SA subject checked against `SUPERUSER_SUBJECTS` allowlist
6. **Robot Mapping:** SA mapped to robot account `quay-system+kube_<namespace>_<sa_name>`
7. **Superuser Grant:** If subject in `SUPERUSER_SUBJECTS`, robot registered as superuser

### Configuration

```yaml
FEATURE_KUBERNETES_SA_AUTH: true

KUBERNETES_SA_AUTH_CONFIG:
  # Kubernetes API server OIDC issuer (auto-discovered in-cluster)
  OIDC_SERVER: "https://kubernetes.default.svc"

  # Expected audience claim (tokens must be created with this)
  EXPECTED_AUDIENCE: "quay"

  # Organization owning SA robot accounts
  SYSTEM_ORG_NAME: "quay-system"

  # Only these SAs can authenticate (also get superuser perms)
  SUPERUSER_SUBJECTS:
    - "system:serviceaccount:quay-operator:quay-operator-controller-manager"
```

### Token Creation

Operators must create tokens with the expected audience:

```bash
kubectl create token <sa-name> --audience=quay
```

Or via projected service account token volume:

```yaml
volumes:
  - name: quay-token
    projected:
      sources:
        - serviceAccountToken:
            audience: quay
            expirationSeconds: 3600
            path: token
```

### Security Considerations

- **Audience Validation:** Always enabled to prevent token replay attacks
- **Subject Allowlist:** Only explicitly configured SAs can authenticate
- **TLS Verification:** Uses in-cluster CA bundle for K8s API server
- **Short-lived Tokens:** Relies on Kubernetes token rotation (bound service account tokens)

## How Kubernetes Service Account Tokens Work

### Bound Service Account Tokens

Kubernetes 1.20+ uses bound service account tokens (KEP-1205) by default. These tokens are:

- **Projected into pods** via volume mounts rather than auto-mounted secrets
- **Audience-bound** to specific consumers (e.g., `quay`)
- **Time-bound** with configurable expiration (default: 1 hour)
- **Object-bound** to a specific pod, preventing use after pod deletion

The kubelet automatically refreshes tokens before expiration. Applications should re-read the token file periodically rather than caching the token value.

```yaml
# Token projection example
volumes:
  - name: sa-token
    projected:
      sources:
        - serviceAccountToken:
            path: token
            audience: quay           # Bound to this audience
            expirationSeconds: 3600  # 1 hour TTL
```

### Token Lifecycle

| Phase | Duration | Description |
|-------|----------|-------------|
| Creation | Instant | Kubelet requests token from API server |
| Valid | ~80% of TTL | Token is usable for authentication |
| Refresh | Before expiration | Kubelet fetches new token, overwrites file |
| Expiration | After TTL | Token rejected by validators |

There is no revocation mechanism for individual tokens. Security relies on short TTLs—if a token is compromised, the exposure window is limited to the remaining TTL.

### OIDC Discovery

Kubernetes API server exposes standard OIDC discovery endpoints, enabling external systems to validate tokens without direct API access:

| Endpoint | Purpose |
|----------|---------|
| `/.well-known/openid-configuration` | OIDC discovery document with issuer and JWKS URI |
| `/openid/v1/jwks` | JSON Web Key Set containing public signing keys |

This allows Quay to validate tokens using only HTTP requests to fetch public keys, without needing Kubernetes API credentials or the TokenReview API.

### JWT Structure and Claims

Kubernetes service account tokens are standard JWTs with three base64url-encoded segments:

```
┌─────────────────────────────────────────────────────────────────┐
│                        JSON Web Token                           │
├─────────────────┬─────────────────────────┬─────────────────────┤
│     Header      │        Payload          │     Signature       │
│   (base64url)   │      (base64url)        │    (base64url)      │
├─────────────────┼─────────────────────────┼─────────────────────┤
│ {"alg":"RS256", │ {"iss":"https://...",   │ RSASSA-PKCS1-v1_5   │
│  "kid":"abc123"}│  "sub":"system:sa:...", │ signature of        │
│                 │  "aud":["quay"],        │ header.payload      │
│                 │  "exp":1234567890}      │                     │
└─────────────────┴─────────────────────────┴─────────────────────┘
                          │
                 eyJhbGci...  .  eyJpc3Mi...  .  SflKxwRJ...
```

#### Key Claims

| Claim | Example | Description |
|-------|---------|-------------|
| `iss` | `https://kubernetes.default.svc` | Issuer (K8s API server) |
| `sub` | `system:serviceaccount:quay-operator:controller` | Subject (SA identity) |
| `aud` | `["quay"]` | Intended audience |
| `exp` | `1704067200` | Expiration timestamp |
| `iat` | `1704063600` | Issued-at timestamp |
| `nbf` | `1704063600` | Not-before timestamp |
| `kubernetes.io/serviceaccount/namespace` | `quay-operator` | SA namespace |
| `kubernetes.io/serviceaccount/name` | `controller` | SA name |

#### JWKS Endpoint

The `/openid/v1/jwks` endpoint returns public keys in JWK format:

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "abc123...",
      "alg": "RS256",
      "use": "sig",
      "n": "<modulus>",
      "e": "AQAB"
    }
  ]
}
```

Token validation:
1. Extract `kid` from JWT header
2. Find matching key in JWKS by `kid`
3. Verify signature using the RSA public key

## Design Details

### Robot Account Naming

ServiceAccounts are mapped to robot accounts using the pattern:

```
<system-org>+kube_<namespace>_<sa-name>
```

Example: `quay-system+kube_quay-operator_quay-operator-controller-manager`

This ensures:
- Unique robot per SA across namespaces
- Easy identification of K8s-originated robots
- Isolation in a dedicated system organization

### Superuser Registration

Robots mapped from `SUPERUSER_SUBJECTS` are dynamically registered as superusers at authentication time. This differs from static `SUPER_USERS` config:

- No config reload required when SA robots are created
- Superuser status tied to authentication, not static config
- Revocation is automatic when SA is removed from allowlist

### JWKS Caching

The Kubernetes OIDC JWKS endpoint response is cached with a configurable TTL (default: 1 hour). On signature verification failure, the cache is invalidated and keys are re-fetched to handle key rotation.

### Demo: Inspecting a Token

Use these commands to explore Kubernetes SA tokens and OIDC endpoints:

```bash
# Create a token with the quay audience
TOKEN=$(kubectl create token default --audience=quay -n default)

# Decode the header (first segment)
echo $TOKEN | cut -d. -f1 | base64 -d 2>/dev/null | jq .
# Output: {"alg":"RS256","kid":"abc123..."}

# Decode the payload (second segment)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq .
# Output: {"iss":"https://kubernetes.default.svc","sub":"system:serviceaccount:default:default",...}

# View the OIDC discovery document
kubectl get --raw /.well-known/openid-configuration | jq .

# View the JWKS (public keys)
kubectl get --raw /openid/v1/jwks | jq .
```

Note: The base64 decode shows the claims but does not verify the signature. Production systems must validate the signature against the JWKS before trusting claims.

## Alternatives Considered

### Static Robot Credentials

**Current approach.** Operators create robot accounts and store credentials in Kubernetes secrets.

Pros:
- Simple, well-understood
- Works with any client

Cons:
- Manual credential rotation
- Long-lived secrets
- Risk of credential leakage

### mTLS with Client Certificates

Use Kubernetes CA to issue client certificates for pods.

Pros:
- Strong authentication
- No bearer tokens

Cons:
- Complex PKI management
- Certificate rotation challenges
- Not natively supported by Kubernetes pods

### Kubernetes TokenReview API

Call Kubernetes API to validate tokens instead of OIDC.

Pros:
- Direct validation with K8s control plane
- Works with all token types

Cons:
- Requires network access to K8s API from Quay
- Higher latency per request
- Tighter coupling to specific cluster

## References

- [Kubernetes Service Account Token Volume Projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection)
- [Kubernetes OIDC Token Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
- [RFC 7519 - JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
