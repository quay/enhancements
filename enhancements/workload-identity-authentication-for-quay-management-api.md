---
title: workload-identity-authentication-for-quay-management-api
authors:
  - "@Marcusk19"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-08-04
last-updated: 2026-08-04
status: provisional
see-also:
  - "https://redhat.atlassian.net/browse/PROJQUAY-12483"
  - "https://redhat.atlassian.net/browse/PROJQUAY-11090"
---

# Workload Identity Authentication for Quay Management API


## Status

*First draft for architecture review. This document captures the proposed boundaries and unresolved
decisions; it is not an implementation plan.*

## 1. Purpose

Enable Kubernetes workloads to obtain scoped Quay Management API OAuth tokens using their
ServiceAccount identity, without storing a shared Quay administrative credential.

## 2. Goals and non-goals

#### Goals

- Authenticate eligible Kubernetes ServiceAccounts to Quay.
- Issue standard Quay OAuth tokens with enforced scopes and normal expiration/revocation behavior.
- Preserve existing human, non-Kubernetes, and programmatic-bootstrap flows.
- Provide auditable success and failure outcomes.

#### Non-goals

- Replacing human authentication.
- Removing programmatic bootstrap in this feature.
- Defining class-level implementation details.

## 3. End-to-end architecture

Workload → Quay Management API: presents a ServiceAccount bearer token and requested access.

Quay → Kubernetes API: validates the token through TokenReview (provisional recommendation).

Quay: resolves the validated identity, applies the selected Quay authorization mapping, and reuses
the existing organization token endpoint/lifecycle.

Quay → Workload: returns a scoped OAuth bearer token, or rejects the request without issuing one.

*Trust boundary: a valid Kubernetes identity is not, by itself, Quay authorization. Quay must apply
an explicit authorization decision before minting a token.*

## 4. Validation approach

*Working recommendation:* Kubernetes API validation using TokenReview, followed by Quay-side
authorization. This keeps Kubernetes as the authority for ServiceAccount token validity and
rotation.

*Alternative for review:* locally validate the JWT using issuer, signature, audience, and claims.
This may reduce API dependency but makes issuer/key-discovery, revocation semantics, and trust
configuration Quay concerns.

## 5. Authorization mapping — open decision

How should a validated identity map to a Quay subject and allowed Management API scopes?

- Explicit configured mapping: cluster/namespace/ServiceAccount → Quay identity + scopes. Strong
  auditability; more configuration.
- Namespace-based defaults: convenient conventions; risk of unintended access.
- Kubernetes RBAC-driven mapping: aligns with Kubernetes permissions; scope translation is complex and
  may couple systems.
- Quay identity per ServiceAccount: clear attribution; introduces identity lifecycle and cleanup
  questions.

### 5.1 Authorization decision: SAR or explicit mapping

The design must choose how a validated Kubernetes ServiceAccount becomes authorized for Quay access.

#### Option A — Kubernetes SubjectAccessReview (SAR)

Quay validates the ServiceAccount with TokenReview, then asks Kubernetes SAR whether the identity
may perform a defined action. This makes Kubernetes RBAC the authorization authority.

**Advantages:** reuses Kubernetes RBAC and avoids introducing a second mapping model.

**Concerns:** Kubernetes verbs/resources do not map naturally to Quay repository, organization, and
**Management API** scopes; it adds another Kubernetes API dependency and call; and it makes Quay
permissions indirect and harder to audit. **SAR** is appropriate only if the product deliberately
defines Kubernetes RBAC as the source of truth for Quay access.

#### Option B — Explicit administrator-controlled mapping (preferred working direction)

Quay validates the ServiceAccount with TokenReview, then looks up an administrator-created mapping
from cluster identity, namespace, and ServiceAccount to a Quay subject and an allow-list of Quay
scopes.

**The mapping must explicitly define:**

- Which cluster or Kubernetes issuer is trusted.
- Namespace and ServiceAccount identity (including audience/issuer constraints where required).
- The Quay subject to use, such as a robot account or service identity.
- The permitted Quay organizations, repositories, operations, or Management API scopes.
- Lifecycle, ownership, audit, and revocation behavior.

#### What is the mapping object?

Preferred for Operator-managed deployments: a dedicated Kubernetes CRD, for example
WorkloadIdentityMapping, managed declaratively by the Quay Operator. The Operator validates the
object and delivers a normalized mapping to Quay; Quay remains responsible for token issuance and
Quay-scope enforcement.

**Other representations to evaluate:**

- Quay-native configuration values: suitable for standalone or non-Kubernetes Quay, but less GitOps-
  and namespace-native.
- A ConfigMap: easy to deploy, but weakly typed and not ideal for validation or lifecycle status.
- A Secret: not appropriate for the mapping itself; it may hold credentials but should not be the
  policy interface.
- An administrative Quay API/UI: useful for standalone administration, but requires a separate
  Kubernetes integration path.

#### Decision guidance

Use SAR only if Kubernetes RBAC is intentionally the policy authority. Otherwise, use an **explicit
mapping**, preferably a Quay Operator-managed CRD for Kubernetes deployments plus an equivalent
Quay-native configuration mechanism for standalone deployments.

## 6. Token issuance

*Working assumption:* reuse the existing organization token CRUD/issuance endpoint and OAuth
lifecycle. Workload identity changes how the caller is authenticated; it does not create a second
token format or lifecycle.

## 7. Compatibility and rollout

- Additive feature, gated by FEATURE_KUBERNETES_SA_BOOTSTRAP and off by default.
- Existing authentication and programmatic bootstrap remain functional when workload identity is
  disabled.
- Existing bootstrap credentials and workload identity may coexist during adoption.
- Migration or deprecation of shared bootstrap credentials is intentionally open.

## 8. Focused security decisions

- Reject unknown, invalid, expired, or untrusted identities before token issuance.
- Enforce requested scopes against the workload’s authorized scopes; never grant broader access by
  default.
- Preserve token expiration, revocation, and audit behavior.
- Record successful issuance and rejected attempts with enough identity and reason context for
  investigation, without logging bearer tokens.
- Define replay and audience requirements for presented ServiceAccount tokens.

## 9. Open questions

- Which authorization mapping model should be adopted?
- Is SubjectAccessReview required in addition to TokenReview, and what exact decision does it provide?
- Where should trust and mapping configuration live, and how is it managed by the Quay Operator?
- If operator managed - what are the defaults?
- What audience, issuer, and cluster identity constraints are required?
- What is the migration and rotation path for existing programmatic-bootstrap credentials?
- Which Kubernetes and non-Kubernetes deployment scenarios are release-supported?
- Audit logs within scope for this feature?

## Review outcome sought

Agree on the trust/validation boundary, authorization mapping direction, endpoint reuse, and rollout
assumptions before implementation design begins.
