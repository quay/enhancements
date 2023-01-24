---
title: Scalable Quota with Layer Dedupication
authors:
  - "@bcaton85"
reviewers:
  - "@dmage"
  - "@kwestpha"
  - "@sleesinc"
approvers:
  - "@dmage"
  - "@kwestpha"
  - "@sleesinc"
creation-date: 2023-01-24
last-updated: 2023-01-24
status: provisional
---

# Overview

When enabling the Quota feature single blobs are counted multiple times in the namespace and repository totals. This is due to the total being summed at the manifest level which does not account for blobs being used in multiple manifests. Instead the total should result from the individual blob sizes contained at the namespace and repository level. This requires re-architecting how the quota totals are calculated since namespaces running in Quay at scale can contain millions of blobs.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Currently Quota creates totals by summing the manifest sizes at the repository and namespace level. This creates an issue where a single blob can by counted multiple times within the total. To get a more accurate count the individual blob sizes will need to be summed at the repository and namespace level. This can only be done by revising the Quota feature to sum these values in a scalable way. The proposal can be broken down into the following stages:
- Calculate the sum of individual blobs at the namespace and repository level
- Calculate the sum of "reclaimable" storage - storage that can be garbage collected
- Display these values to the user in an intuitive way (via API and UI)

## Motivation

This allows users to place Quota's on the namespace and repository levels based on the actual usage of storage by Quay.

### Goals

* Total storage usage at the repository and namespace levels that account for re-used blobs
* Include blobs contained within manifest lists as a part of storage calculation
* Ability to view the amount of storage that is reclaimable
* Ability to run the feature at scale without issue

### Non-Goals

TBD

## Open Questions

* How to re-calculate totals between restarts?
* What is the effect of adding 2 reads and 2 writes to every blob push?
* How to we present to users that the pre-existing sum is still running?
* How do we enforce Quota if the pre-existing sum is still running?
* How do we ensure the total is accurate between restarts?

## Design Details

#### Context

Quota works by totalling the size of all the blobs under a manifest then writing that total out to a `repositorysize` table. The size is retrieved for reporting and quota enforcement needs. When a push or delete occurs the size is recalculated and written back out to the repository size table. A similar mechanism happens at the namespace level.

If we wanted to get a total of the _individual blobs_ within a repository an intuitive approach is to sum the blob sizes within the following relation. The `manifestblob` table maps manifests to blobs and the `imagestorage` table represents the individual blobs.

and for namespaces:

Directly replacing the current calculation method with summing unique entries in the `imagestorage` table gives us the desired effect, including the size of manifest lists, but does not work at scale. The size of the `manifestblob` and `imagestorage` tables can be in the millions for a single organization. While the sum can be done in milliseconds with thousands of blobs it takes minutes for millions of blobs.

A background job can be used to sum the total asynchronously. The challenge being scale again. When ever a blob is created/deleted the sum is now stale and will need to be recalculated. The time it takes to compute the reported value (time in queue+actual computation) will cause drift from the real value, allowing namespaces/repositories to go beyond their quota limits. 

#### Using a Running Total

Another approach is to keep a running total at the namespace/repository level. As blobs (represented as rows in the `imagestorage` table) are created and deleted the value from `imagestorage.image_size` is added/subtracted from a table holding the total size (`namespacesize.size`/`repositorysize.size`). So as blobs are created/deleted we're able to have an accurate size without recalculating the size of individual blobs.

This does not cover the total size of blobs that already exist in the namespace/repository. This can run as a background job that calculates the current total of each namespace/repository. Another value can be used to indicate that the summation of pre-existing blobs is running (`namespacesize.up_to_date`). When completed it can be added to the running total to get the current size of the namespace/repository.

**(The following section is under review and revision)**
All blobs must be apart of either the running total or pre-existing summation. Therefore there needs to be a way to bucket the blobs between the two. This can be accomplished with a timestamp of when the pre-existing summation has started. Any blob existing before the timestamp will be a part of the pre-existing summation and any after will be included in the running total.

#### Calculating Reclaimable Space

TBD

### Constraints

* Large repositories and registries with many namespaces require more time to calculate. This means Quota and reporting will not be available immediately as the pre-existing sum is running.

### Risks and Mitigations

* Requires modifictions to critical paths of Quay functionality, introducing impact to core features

### Test Plan

TBD
