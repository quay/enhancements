---
title: TLS Managed Component
authors:
  - "@alecmerdler"
reviewers:
  - "@jonathankingfc"
approvers:
  - "@jonathankingfc"
  - "@alecmerdler"
creation-date: 2021-04-27
last-updated: 2021-04-27
status: implementable
---

# tls-managed-component

Quay has traditionally preferred to serve HTTPS traffic directly from the 
container, rather than having TLS terminated by some external service.
This proposal is to make external TLS termination the opinionated method
for the Quay Operator.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions 

> 1. Is it acceptable to security-minded users that the `Route` resource 
contains TLS private key in plaintext on the `spec` block?
> 2. How does OCP handle the `Ingress` resource?

## Summary

When running Quay on Kubernetes/OpenShift, it is possible to terminate HTTPS
connections at some level outside the pod/container, using the `Ingress`
or `Route` API.

## Motivation

Many users want to take advantage of the OpenShift-included PKI infrastructure
to secure their Quay endpoint with TLS. This is currently not possible,
because the Quay Operator sets up TLS termination at the Quay pod level by using a `Route` with TLS passthrough, and expects 
the required TLS cert/key pair to be part of the config bundle `Secret` which
is mounted into the Quay app pod.

In the current implementation, if a TLS cert/key pair is not provided as 
part of the config bundle `Secret`, then a self-signed pair is generated
and used instead. This commonly violates customer requirements.

### Goals

* Creating a zero-config `QuayRegistry` on OpenShift will be secured with TLS provided by the cluster.
* Creating a `QuayRegistry` which includes TLS cert/key pair in the `spec.configBundleSecret` will be secured using the given cert/key pair.
* Separate TLS certificate handling from the `route` component to a new managed component: `tls`.

### Non-Goals

* Quay or the Quay Operator provide their own PKI infrastructure.

## Proposal

Make edge TLS termination the opinionated default when a `QuayRegistry` is 
reconciled by the Quay Operator. This involves the following changes to 
how the Operator currently behaves:

* Ensuring the following values in the generated `config.yaml`:

```yaml
EXTERNAL_TLS_TERMINATION: true
PREFERRED_URL_SCHEME: https
```

* The files `ssl.cert`/`ssl.key` must not be present in `/conf/stack` in the Quay
container, or else the [wrong NGINX configuration will be generated](https://github.com/quay/quay/blob/master/conf/init/nginx_conf_create.py#L72).

* When using a managed `route` component, then the Operator will create a `Route`
with `termination: edge`. If the user provided a `ssl.cert`/`ssl.key`, they will
be added directly to the `Route` object (`certificate`/`key` fields).

### Risks and Mitigations

* In-cluster communication between Quay and its dependencies will not use TLS
(unless being accessed using Quay's external hostname rather than internal Kubernetes hostname).

## Design Details

### Test Plan

| `route` | `tls` | TLS cert/key pair provided | Expected result |
|---|---|---|---|
| Managed | Managed | No | Edge `Route` with default wildcard cert |
| Managed | Managed | Yes | Edge `Route` with default wildcard cert (Ignore provided TLS) |
| Managed | Unmanaged | No | Error, TLS cert/key pair must be provided |
| Managed | Unmanaged | Yes | Edge `Route` with provided TLS |
| Unmanaged | Unmanaged | No | Do nothing, Quay expects HTTP traffic |
| Unmanaged | Unmanaged | Yes | Do nothing, Quay expects HTTP traffic |
| Unmanaged | Managed | No | Error, `tls` component can only be used with `route` |
| Unmanaged | Managed | Yes | Error, `tls` component can only be used with `route` |

### Upgrade / Downgrade Strategy

**QuayEcosystem upgrade**

The logic to migrate a `QuayEcosystem` to a `QuayRegistry` is still present
in the controller codebase. 

For `QuayEcosystems` which have `spec.quay.externalAccess.tls.termination`
set to `edge`, the logic will be changed to allow migration of the `Route`.
If `spec.quay.externalAccess.tls.secretName` is set, we will fetch the TLS 
cert/key pair from that `Secret` and store it in the `configBundleSecret`, 
so that the `QuayRegistry` controller will handle it as normal.

**QuayRegistry upgrade**

If no user-provided TLS cert/key pair in `configBundleSecret`, the Operator
will remove the generated self-signed pair (from previous implementation) 
from the `configBundleSecret`, and the `Route` will be switched to `edge`
termination using the cluster wildcard cert.

If user-provided TLS cert/key pair in `configBundleSecret`, the Operator
will copy it into the `Route` and switch it to `edge` termination.

### Version Skew Strategy

Quay's Python codebase already supports `EXTERNAL_TLS_TERMINATION: true`, 
so there are no issues with version skew and this component.

## Implementation History

* 2021-04-27 Initial proposal

## Drawbacks

* It is now more difficult to provide your own certs to use with a custom 
hostname, but this will be improved upon in the future with integrations with
PKI infrastructure at the Kubernetes API level (the `Certificates` API).

## Alternatives

* Continue to terminate TLS at the Quay container
* Use the `reencrypt` termination strategy, which would use both the OCP-provided TLS and the generated self-signed certs in the Quay pod
