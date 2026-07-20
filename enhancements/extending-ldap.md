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


### Implementation

#### Superusers

In addition to current superuser capabilities, this implementation will add the ability for superusers to access contents from outside its permission scope. This is a very specific use case and would only be enabled if the given FEATURE flag is set. This would allow superusers, on top of taking ownership of other namespaces, read, write, and delete contents from other namespaces, without breaking backward compatibility (i.e exising installations should be able to opt-in to get this behavior).

In terms of permission, access to resources not owned by the user would be granted based on whether or not that user has the existing superuser grant (scope) or not. This means that superusers would essentially bypass standard authentication in these cases. TODO: security implications.

#### Restricted Users

Restricted users are a subset of Quay users whose capabilites will be a subset of current regular Quay users (excluding superusers). Some limitations we're going to impose are:
- Inability to create organizations
- Inability to write (Feature flagged. Should this be enforced on owner's namespace, or just enforced through quota?)
- Feature flag/whitelist to default new users as restricted (when not using LDAP, or other federated provider)

Like superusers, restricted users are defined in LDAP using a filter or group. When LDAP is not used, unlike how superusers are defined in a list in Quay's configuration, restricted users are defined based on a provided whitelist (i.e users are defined as restricted unless specified in that configuration list).

When that list is not set, then this feature should implicitly not be enabled (only in the case where LDAP is not used.

#### Configuration

`FEATURE_SUPERUSERS_FULL_ACCESS`: Whether to grant superusers full access no namespaces not owned
`FEATURE_RESTRICTED_USERS`: Whether to enable restricted users
`RESTRICTED_USERS_WHITELIST`: Defines the list of users who are not limited by the restricted users' constraints (i.e regular Quay user)
`LDAP_SUPERUSER_FILTER`: Filter for selecting ldap superusers. Relative to the LDAP_USER_FILTER.

