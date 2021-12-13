---
title: Quota Management and Enforcement on Quay
authors:
  - "@kwestpha"
  - "@sleesinc"
  - "@sdadi"
reviewers:
  - "@kwestpha"
  - "@sleesinc"
  - "@sdadi"
approvers:
  - "@kwestpha"
  - "@sleesinc"
  - "@sdadi"
creation-date: 2021-12-03
last-updated: 2021-12-03
status: implementable
---

# quota-management-and-enforcement

Currently, Quay users have very little control to contain unbounded growth of storage consumption of the registry.
This proposal is to report storage consumption and contain registry growth based on the configured storage quota limits.  

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

 > 1. Do we have any here?

## Summary

There are no limitations on a Quay organization in terms of their storage consumption. This is a three-phase proposal that
aims to contain and provide a smart solution to tackle storage related problems for Quay customers.   
In the first phase i.e, Quota Reporting: a super admin can see the storage consumption of all of their organizations, and a user can see the storage consumption of their organization.
In the second phase i.e, Quota Management: a superuser can define soft and hard checks for Quay users.
A soft check implies that the Quay user will be notified if the storage consumption of an organization reaches the configured threshold.
A hard check implies that the Quay user will not be allowed to push to the registry once the storage consumption reaches the set limit.
In the third phase, a quay admin can configure auto pruning of the images based on when they were last used to free up quota.

## Motivation

Users can create new organizations and repositories in their home organizations without any practical storage limitation.
On-prem Quay customers are subject to capacity limits of their environment, specifically backend storage consumption.
No knowledge about their storage consumption makes it hard for a service owner of a Quay registry to define SLAs and maintain a budget.

### Goals

* A Quay superuser can see the overall storage consumption of the entire registry i.e., an overview of all organizations depicting the aggregate storage consumption of images in an organization.
* The storage consumption per repository and organization (for super user) will be exposed on an endpoint (Like: /api/v1/repository/<namespace>/<reponame>/limits) to see their current consumption.
* A Quay user can define soft quota limit on an organization that will trigger a warning when the set limit has reached.
* A Quay user can configure alerts that will allow Quay to send notifications when set storage threshold has reached.
* A Quay user can define hard quota limit on an organization which will not allow them to push any more images when set threshold has reached.

### Non-Goals

* We will not be implementing shared layer calculation i.e., if a user pushes two different versions of an image and there are some common layers between the two,
the storage consumption is calculated independently for each of the common layers.

## Design Details

Initial work is to add Quota Reporting on organizations i.e., providing Quay users with their storage consumption data on a Quay organization.
In the next steps, we will add Quota Enforcements i.e., soft and hard checks that notify or restrict Quay users from pushing images to registry.

The expected design flow is depicted as below:
![](https://user-images.githubusercontent.com/11522230/145486007-c88b980c-114a-44bc-a7e0-aa13a266cc79.png)
NOTE: The boxes with the black border show the current flow, and the boxes with the green border show what needs to be implemented to support this feature. Design credits: @kwestpha

The table RepositorySize (to be created) stores the storage consumption(in bytes) of a Quay repository within an organization. The sum of all repository sizes for an organization,
defines the current storage size of a Quay organization.
When an image push is initialized, the user's organization storage is validated to check if it is beyond the configured Quota limits.
If yes, for a soft check, user is notified and, for a hard check, the push is stopped. If storage consumption is within configured Quota limits,
the push happens as expected.

Images are uploaded to the Quay registry as chunks. Each of these chunk upload triggers, the next flow.
The size of each image chunk is stored a temporary table, blobupload that has a foreign key reference to the Repository table.
This size of every chunk upload is added to the current repository size that calculates a temporary repository size. This temporary size is validated to be
within the configured Quota limits. If this temporary size reaches beyond the configured Quota limit, an exception is raised.

Images manifest deletion follows the same flow as on Quay where the link between associated tags and the manifest are deleted. One addition to this,
is, after the manifest is deleted, the repository size is recalculated and updated in the RepositorySize table.
This is depicted below:
![](https://user-images.githubusercontent.com/11522230/145113938-7a8e3fcf-693a-488e-a711-bcf116e04446.png)

### Constraints

A few caveats with doing the calculations on push vs a background worker is that existing images will not have any usage data.
This means that a backfill for all existing repositories will be required before the feature is enabled.
This also means that this calculation becomes part of a push's critical path, otherwise the usage data could possibly drift.

### Risks and Mitigations

* We want to avoid creating a bottleneck in the push flow for quay that could impact usability of the product.
* We need to avoid increasing database pressure that could create a bottleneck and impact the usability of the product.
* We need to ensure that the design of the feature does not impact users that do not choose to use this product feature.

### Test Plan

* Implement Quota Reporting on organization level.
* Implement Quota Management for soft and hard checks on an organization level.
* TBD: Implement Auto Pruning

### Implementation History

* 2021-12-03 Initial proposal
