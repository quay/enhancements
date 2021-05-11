---
title: Quota and Enforcement for Quay
authors:
  - "@kleesc"
reviewers:
  - "@alecmerdler"
approvers:
  - "@alecmerdler"
creation-date: 2021-05-11
last-updated: 2021-05-11
status: provisional
---

# quota-and-enforcement-reporting (1)

This proposal is to add the ability to view/allocate resource usage at different levels
(namespaces/orgs/users, repositories) and enforce quotas.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Various customers are running Quay in large deployments providing Quay as a service to several internal and external customers. Many of those want to run some kind of showback / chargeback for Quay (reporting) over (soft/hard) quotas.

## Motivation

Provide Quay customers/admins reporting on resource usage, and eventually some kind of
showback/chargeback feature on soft/hard quotas.

### Goals

- Allow admins to view resource usage for namespaces and repos
- Prevent users from taking too much resources (network, storage, repos, ...)
- "Charge" abusers

### Non-Goals

As the first step of quota management, only usage/quota reporting should be implemented.
Any quota (soft or hard) enforcement is to be implemented subsequently based on the work of
this proposal.

### Implementation Details/Notes/Constraints

The resource tracking system will need to be based on multiple resources:
- Database's state for repositories, tags, namespaces, ...
- The logs model for network ingress/egress
- Some sort of aggregation to summarize large amounts of data (e.g storage used)
  This will likely require some optimization (e.g calculating the delta over time; rolling sum)

#### Network bandwidth
TODO

#### Storage

Calculating a large namespace's total blobs could be massive.
Who gets "charged" for shared layers (charging everyone is the simpler solution for a consistent measurement)

##### Possible implementations
- Add a "size" row to the repository table and update that as images get pushed/GC'ed.
  This would require a very large backfill for existing repos.
  Denormalized data (as this data already available in the storage table).
  Possible drifting, if somehow an update fails.
- Have a worker aggregate the size in a separate table, by joining the ManifestBlob and ImageStorage tables for a repo, with a last updated timestamp.

### Risks and Mitigations

Added load on the database, since that's where most of the required data will be aggregated from
(this will be especially true when trying to aggregate large amount of data for storage calculations).
Any new models/tables, might need to be denormalized to decouple them from the existing ones,
in order to reduce the need for more complex joins.

How to deal with workarounds around limits. e.g Making a mono repo to get around repo limits.

## Design Details

### Usage/quota reporting (Phase 1)
TODO

### Soft quota enforcement (Phase 2)
TODO

### Hard quota enforcement (Phase 3)
TODO

### Test Plan
TODO

### Upgrade / Downgrade Strategy

Any new data model changes should either be additive to the existing tables (adding new columns only) or denormalized for performance reasons.
This means any migrations (new tables or new columns) should not cause any downtime when upgrading, or rolling back the schema.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

* Added load on the database, since that's where most of the required data will be aggregated
  from
