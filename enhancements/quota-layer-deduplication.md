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
creation-date: 2023-02-10
last-updated: 2023-02-10
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

Currently Quota creates totals by summing the manifest sizes at the repository and namespace level. This creates an issue where a single blob can by counted multiple times within the total. To get a more accurate count the individual blob sizes will need to be summed at the repository and namespace level. The implementation can be broken down into the following areas of work:
- Addition: Ensuring only unique blobs are counted to the namespace/repository totals
- Subtraction: Ensuring the deletion of only unique blobs are subtracted from the namespace/repository totals
- Backfill: Ensuring pre-existing unique blobs before the feature has been enabled are counted towards the namespace/repository totals
- Reclaimable space: Calculating the estimated reclaimable space within a namespace/repository

## Motivation

This allows users to place Quota's on the namespace and repository sizes based on the actual usage of storage by Quay.

### Goals

* Total storage usage at the repository and namespace levels that account for re-used blobs
* Include blobs contained within manifest lists as a part of storage calculation
* Ability to view the amount of storage that is reclaimable
* Ability to run the feature at scale

### Non-Goals

TBD

## Open Questions

- Are we able to get the query for reclaimable space down to a reasonable duration?
- Need to investigate changes required to proxy pull through feature

## Design Details

#### Context

Quota works by totalling the size of all the blobs under a manifest then writing that total out to a `repositorysize` table. The size is retrieved for reporting and quota enforcement needs. When a push or delete occurs the size is recalculated and written back out to the repository size table. A similar mechanism happens at the namespace level.

If we wanted to get a total of the _individual blobs_ within a repository an intuitive approach is to sum the blob sizes within the following relation. The `manifestblob` table maps manifests to blobs and the `imagestorage` table represents the individual blobs.
![image](https://user-images.githubusercontent.com/22058392/214327014-4d831c92-b482-4a7c-b342-232becdfe033.png)

and for namespaces:
![image](https://user-images.githubusercontent.com/22058392/214327032-235650ec-c905-48c6-9eb0-ce6e9f00f574.png)

Directly replacing the current calculation method with summing unique entries in the `imagestorage` table gives us the desired effect, including the size of manifest lists, but does not work at scale. The size of the `manifestblob` and `imagestorage` tables can be in the millions for a single organization. This requires more time than can be spent in a single request.

A background job can be used to sum the total asynchronously but will still have issues at scale. When ever a blob is created/deleted the sum is now stale and will need to be recalculated. The time it takes to compute the reported value (time in queue+actual computation) will cause drift from the real value, allowing namespaces/repositories to go beyond their quota limits. 

#### Solution: Using a Running Total

Another approach instead of recalculating the total each time is to keep a running total of the namespace/repository sizes. We can split the calculation of blobs in two phases, backfill and inflight. Backfill will count each pre-existing blob in the registry and the inflight will keep a running total of the blobs in the future. All blobs can be accounted for by using the start time of the backfill.

**Backfill** A table `namespacesize` will be added with the columns `namespace_id, size_bytes, backfill_start_ms, backfill_complete`. The backfill runs within an asyncronous worker that discovers namespaces/repositories to run the backfill for. The backfill worker then executes the following steps:
1. Discover namespaces/repositories to calculate backfill total for
    - Either on demand through queue or background by searching the repository/namespace tables
2. For each namespace/repository insert the start time of the total into the database under `backfill_start_ms`
3. Run the total for the namespace/repository using the following queries which account for deduplicating blobs.
      ```python
      # Namespace
      derived_ns = (
          ImageStorage.select(ImageStorage.image_size)
          .join(ManifestBlob, on=(ImageStorage.id == ManifestBlob.blob))
          .join(Repository, on=(Repository.id == ManifestBlob.repository))
          .where(Repository.namespace_user_id == namespace_id)
          .group_by(ImageStorage.id)
      )
      total = ImageStorage.select(fn.Sum(derived_ns.c.image_size)).from_(derived_ns).scalar()
      ```
      ```python
      # Repository
      derived_ns = (
          ImageStorage.select(ImageStorage.image_size)
          .join(ManifestBlob, on=(ImageStorage.id == ManifestBlob.blob))
          .where(ManifestBlob.repository == repository_id)
          .group_by(ImageStorage.id)
      )
      total = ImageStorage.select(fn.Sum(derived_ns.c.image_size)).from_(derived_ns).scalar()
      ```
4. Write out the total to `size_bytes` and set `backfill_complete` to `true`

**Inflight** To quickly get the new total without re-running the entire sum we can add and subtract the individual blob sizes as manifests are deleted and created. The workflow for addition and subtraction can be as follows:

Addition
1. Recieve the manifest that is being created along with it's associated blobs
2. Check if the current time is after `backfill_start_ms`, exit if not
    - If the backfill has already started any future blobs need to be accounted for by inflight
3. For each blob, check if it exists in the namespace already
    - If it does not this means this is the first occurence of this blob and it should be added to the total
4. For each blob, check if it exists in the repository already
    - If it does not this means this is the first occurence of this blob and it should be added to the total

Subtraction
1. Recieve the manifest that is being created along with it's associated blobs
2. Check if the current time is after `backfill_start_ms`, exit if not
    - If the backfill has already started any future blobs need to be accounted for by inflight
3. For each blob, check if it exists in the namespace
    - If it does not this means this is the last occurence of this blob and it should be subtracted to the total
4. For each blob, check if it exists in the repository
    - If it does not this means this is the last occurence of this blob and it should be subtracted to the total

Using these two totals allows for immediate updates to the namespace/repository sizes at scale. Using a running total also has the risk of becoming out of sync. By setting `backfill_start_ms` to `null` and `backfill_complete` to `false` for a specific namespace/repository the backfill will run again and return with the accurate totals.


**Reclaimable space**

TBD


### Constraints

- During rolling deployments previous instances may delete blobs that would go unaccounted by both the backfill and inflight totals. For this reason the feature can only begin totalling until after the rolling deployment has finished.

### Risks and Mitigations

* Requires modifictions to critical paths of Quay functionality, introducing impact to core features

### Test Plan

TBD
