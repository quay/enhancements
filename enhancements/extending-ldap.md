---
title: Proposal to extend user definitions in LDAP
authors:
  - "@kleesc"
reviewers:
  - "@bcaton85"
approvers:
  - "@bcaton85"
creation-date: 2022-07-08
last-updated: 2022-07-08
status: implementable
---

# Extending users groups in LDAP

## Summary

Currently, Red Hat Quay supports the use of LDAP as an identity
provider for Quay. This means when setup, users can login/register to
Quay using their LDAP credentials setup by their administrators. Other
Quay features based on LDAP includes team membership syncing with
groups.

As part of Quay 3.8, we are looking at extending this to support
different type of users based on LDAP group/filters.  These would
specifically be targetted towards enterprise users, as opposed to
Quay.io.

## Open Questions

- TODO

### Goals

- Define different set of users based on an external identity provider (non-database)
- Still backwards compatible with the current state

## Design Details

### Superusers

Currently, when superusers are enabled, the list of users with such
access is defined at runtime, in Quay's configuration file. This can
get tedious to maintain when large number of superusers are
required. It also means having multiple set of users to maintain, in
LDAP AND in Quay.

This proposal is to add the possibility of maintaining that same list
of users, in LDAP itself, either based on some extra LDAP group or
filters. The main benefit of such feature is that system administrator
would now only have to worry about their LDAP deployment, and not the
Quay configuration when on-boarding and off-boarding members.

### Restricted users

Similarly, we can define another different set of LDAP users, with
restricted access to content. One of the use case would be to give
temporary, or read-only access to such user (for auditing purposes,
for example).

Another use case would be to limit who can or cannot create
organization, as it is currently not enforced at all. (i.e any user is
able to create and squat on a organization's namespace).

### Api access constraints based on authentication group

Based on these extra groups/filters that allows Quay to select
distinct users groups from the identity provider, we can extend the
federated user identity interface (`data/users/`) to select a specific
set of users. For example, being able to return a list of superusers
from an LDAP deployment.  Another example, for a Quay deployment using
LDAP, Quay could limit creation of organizations to a specific set of
user group based on some LDAP constraint. Similarly, we could limit
(or grant) access to Quay's api or content based on another similarly
defined LDAP entity.

## Constraints

- This proposal would only be for deployments making use of a federated identity with Quay (LDAP, keystone, ...)

## Risks and Mitigations

- The implementation of this proposal should not affect or block any of Quay.io's deployments. Any features, when implemented should be feature flagged, as not every deployment uses a federated identity service like LDAP.

## Implementation History

- 2022-07-08 Initial proposal
