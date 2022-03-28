---
title: Quay as a Proxy Cache
authors:
  - "@fmissi"
  - "@sleesinc"
  - "@sdadi"
reviewers:
  - "@fmissi"
  - "@sleesinc"
  - "@sdadi"
approvers:
  - "@fmissi"
  - "@sleesinc"
  - "@sdadi"
creation-date: 2021-12-10
last-updated: 2021-12-10
status: implementable
---

# Quay as a cache proxy for upstream registries

Container development has become widely popular. Customers today rely on container images from upstream registries(like docker, gcp) to get desired services up and running.
Registries now have rate limitation and throttling on the number of times users can pull from these registries.
This proposal is to enable Quay to act as a pull through cache where, once pulled images are only pulled again when upstream images have been updated.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

 1. Will the check against the upstream registry count against an upstream rate limit?
 A: No. We use HEAD requests to check for new versions of an image, and HEAD requests do not count against rate limit.
 2. Currently, How can we track when an image was last used (to implement eviction based on LRU images)?
 3. How will time machine work on expired images?
 A: Time machine stops images from being garbage collected within a time period. As long as an image is still around and is up-to-date with the upstream registry, Quay will continue serving it.

## Summary

Dependencies on container images have increased tremendously with the adoption of container driven development. With the introduction of rate limits
on popular container registries, Quay will act as a proxy cache to circumvent pull rate limitation from upstream registries. Adding a cache will also
accelerate pull performance as images are pulled from cache rather than upstream dependencies. Cached images are updated only when the upstream
image digest differs from cached image.

### Goals

* A Quay user can define and configure(credentials, expiration) via app, an organization in Quay that acts as a cache for a specific upstream registry.
* A Quay super can leverage storage quota of an organization to limit cache size. This means when cache size reaches its quota limit,
  images from cache are evicted based on LRU.
* A proxy cache organization will transparently cache and stream images to client. The images in the proxy cache organization should
  support the default expected behaviour (like security scanner, time machine, etc) as other images on Quay.
* Given the sensitive nature of accessing potentially untrusted upstream registry all cache pulls needs to be logged (audit log).
* A Quay user can flush the cache to eliminate excess storage consumption.
* Robot accounts can be created in the cache organization as usual, their RBAC can be managed for all existing cached repositories at a given point in time.
* Provide metrics to track cache activity and efficiency (hit rate, size, evictions).

### Non-Goals

* In the first phase, configuring a cache proxy organization, caching upstreaming images, and quota management on cached repositories is the target.
  Other goals will be implemented subsequently based on the work of this proposal.
* ~~Cached images are read-only, which means that images cannot be pushed into Proxy Cache Organization.~~ This is not part of the tech preview - proxy orgs can still receive pushes.

## Design Details

The expected pull overview is depicted as below:
![](https://user-images.githubusercontent.com/11522230/145866763-58f44c94-839b-4edb-a95b-b9c3648cf187.png)
Design credits: @fmissi

A user initiates a pulls of an image(say a `postgres:14` image) from an upstream repository on Quay. The repository is checked to see if the image is present.
1. If the image does not exist, a fresh pull is initiated.
  * The user is authenticated into the upstream registry. Authetication is not a requirement. For public repositories, docker hub supports anonymous pulls.
    But, for private repositories, user should be authenticated.
  * The layers of `postgres:14` are pulled.
  * The pulled layers are saved to cache and served to the user in parallel.
  * This is depicted as below:
    ![](https://user-images.githubusercontent.com/11522230/145871778-da01585a-7b1b-4c98-903f-809c214578da.png)
    Design credits: @fmissi

2. If the image exists in cache:
  * A user can rely on Quay's cache to stay coherent with the upstream source so that I transparently get newer images from the cache
     when tags have been overwritten in the upstream registry immediately or after a certain period of time.
  * If the upstream image and cached version of the image are same:
    * No layers are pulled from the upstream repository and the cached image is served to the user.

  * If the upstream image and cached version of the image are different:
    * The user is authenticated into the upstream registry and only the changed layers of `postgres:14` are pulled.
    * The new layers are updated in cache and served to the user in parallel.
  * This is depicted as below:
    ![](https://user-images.githubusercontent.com/11522230/145872216-31350e08-6746-4e34-aebf-e59a7bf6b372.png)
    Design credits: @fmissi

3. If user initiates a pull when the upstream registry is down:
  * If the pull happens within the configured expiration period, the image stored in cache is served.
  * If the pull happens after the configured expiration period, the error is propagated to the user.
  * This is depicted as below:

    ![image](https://user-images.githubusercontent.com/444856/160366569-dfb816dc-1bfd-4ec3-9e4e-08fe892b1e29.png)

    Design credits: @fmissi

A quay admin can leverage the configurable size limit of an organization to limit cache size so the backend storage consumption remains predictable
by discarding images from the cache according to least recently used frequency.
This is depicted as below:
![](https://user-images.githubusercontent.com/11522230/145884935-df19297f-96b5-4c1c-9cdc-e199e04df176.png)
Design credits: @sdadi

A user initiates a pulls of an image(say a `postgres:14` image) from an upstream repository on Quay. If the storage consumption of the organization
is beyond the configured size limit, images in the namespace are removed based on least recently used, to make space for `postgres:14` to be cached.

### Constraints

* If size limit is configured in a proxy cache organization, and say an org is set to a max size of 500mb and the image the user wants to pull is 700mb.
  In such a case, the image pulled will be cached and will overflow beyond the configured limit.

### Risks and Mitigations

* The cached images should have all properties that images on a Quay repository would have.
* When the client wants to pull a new image, Quay should reuse already cached layers and download from the upstream repository only new ones.

### Implementation Plan

* Pull a fresh image and check that layers are saved to cache.
* Pull an outdated image and check that only changed layers are saved to cache.
* Implement configuring of a proxy cache organization.
* Implement quota(configurable size limit) management (blocked by [quota mangement feature](https://issues.redhat.com/browse/PROJQUAY-302))

### Implementation History

* 2021-12-13 Initial proposal
* 2022-03-28 Answer a couple open questions, update non-goals
