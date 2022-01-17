---
title: Unmanaged Clair Database
authors:
  - "@jonathankingfc"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2022-01-11
last-updated: 2022-01-14
status: provisional
---

# unmanaged-clair-database

Currently, the operator allows only Quay's database to be unmanaged. There is business need to make Clair's database unmanaged as well, in the same manner as Quay's database now can be set to unmanaged, while still retaining Clair as a managed component. This also requires that a user has the ability to set a custom clair configuration (specifically, a custom database connection string)

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

> 1.  Should the clair config be fully customizable, or should it be limited to specific fields such as the DSN?

## Summary

A user should have the ability to provide a custom clair configuration for an unmanaged clair database. The unmanaged configuration must be valid and clear error messages must be present in the case that the config is invalid.

There are two steps required for this implementation:

- A user has the ability to provide a custom clair config as a secret reference in the CRD
- A clairdatabase component should exist in the CRD

## Motivation

There are several scenarios in which a user of the Quay Operator might want to use an unmanaged database. For example, to allow for the operator to work in a Geo-Replicated environment, multiple instances of the operator must be able to talk to the same database. In the current implementation, Clair can only talk to its managed database inside of a single cluster. Additionally, a user might have a requirement for an HA clair database that exists outside of a cluster.

### Goals

- A `clairdatabase` component should be configurable in the Operator CR.
- An unmanaged `clairdatabase` component should check for the existence of a custom clair configuration file
- The Operator should validate the custom configuration provided when the component is set to unmanaged (the config must fit the clair schema, and the database connection must be working)
- When `clairdatabase` is marked as unmanaged, any `clair-postgres` pods should be stopped and or not provisioned at all

### Non-Goals

- The setup of an unmanaged Clair database is the responsibility of the user and is out of the scope of the operator

## Proposal

The Operator should allow for a user to provide their own Clair configuration in the following manner.

In the provided `spec.configBundleSecret`, an additional field called `clair-config.yaml` should be accepted. An example secret might look like the following:

```
apiVersion: v1
kind: Secret
metadata:
  name: config-bundle-secret
  namespace: quay-enterprise
data:
  config.yaml: <base64 encoded Quay config>
  clair-config.yaml: <base64 encoded Clair config>
```

Once the secret has been created, a user should also be able to unmange the `clairdatabase` component. An example CR might look like the following:

```
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: quay
spec:
  components:
  - kind: clairdatabase
    managed: false
  configBundleSecret: config-bundle-secret

```

The provided clair-config.yaml would be mounted into the Clair pod, and any fields not provided should be automatically populated with defaults using the Clair config module.

### Implementation Details/Notes/Constraints

One implementation detail to note is that a managed Clair component should consume a custom `clair-config.yaml` as long as it is provided. This is similar to how the base Quay component can be customized with a `config.yaml` despite always being a managed component. However, the clair config must be in sync with the state of the `clairdatabase` component. For example, if a user chooses a managed `clairdatabase` component, the `clair-config.yaml` should not contain custom DSN values.

The database connection can be validated using the config-tool module.
See here: https://github.com/quay/config-tool/blob/171819fa5bca7b3a55be146766abd453dcab43b1/pkg/lib/shared/validators.go#L458.

### Risks and Mitigations

Allowing for further configuration increases the likelihood of incorrect configurations. The operator must ensure that the configuration is in fact valid, and report any incorrect configurations. To mitigate this, the Quay Operator documentation should provide a reference to the Clair documentation on how to set up a database and attach this to the `clair-config.yaml`.

## Design Details

### Test Plan

```
| `clair` | `clairdatabase` | Clair Config with DSN Provided | Expected result |
|---|---|---|---|
| Managed | Managed | Yes | Error, managed `clairdatabase` should not contain a DSN |
| Managed | Managed | No | No Error, this is a fully managed Clair instance |
| Managed | Unmanaged | No | Error, Clair config must contain DSN |
| Managed | Unmanaged | Yes | No Error, Clair will use the DSN provided in the custom config |
| Unmanaged | Unmanaged | No | Error, unmanaged Clair should not have a clair config |
| Unmanaged | Unmanaged | Yes | Error, unmanaged Clair should not have a clair config |
| Unmanaged | Managed | No | Error, unmanaged Clair should not have a clair config |
| Unmanaged | Managed | Yes | Error, unmanaged Clair should not have a clair config |
```

This matrix can be tested using unit tests.

In addition to this, e2e tests should be implemented inside a real cluster. This can be done using the `kuttl` tool. A PR has been opened to add the `kuttl` e2e tests to the operator and can be found at https://github.com/quay/quay-operator/tree/kuttl

### Upgrade / Downgrade Strategy

Since previous versions of Quay do not contain a `clair-config.yaml` in the `config-bundle-secret`, and do not have a customizable `clairdatabase` component, upgrades should not affect the behavior of any `QuayRegistry`. After an upgrade, the operator should automatically set `clairdatabase` to be a managed component, and the default generation of the clair config should proceed as normal. In the case that the previous installation has Clair set to unmanaged, the `clairpostgres` component will be set to unmanaged as well.

## Implementation History

- 2022-01-13 Initial proposal

## Drawbacks

The main drawback of the supporting a more advanced configuration is the potential for increased user error.

## Alternatives

One alternative in the API change is to keep the `clair-config.yaml` in a different secret from the `config-bundle-secret`. This might look something like this:

```
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: quay
spec:
  configBundleSecret: config-bundle-secret
  clairConfigBundleSecret: clair-config-bundle-secret
```

The internal implementation of this would look the same as in the original proposal as this is just a change in how the user provides a custom clair config.
