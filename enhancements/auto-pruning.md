---
title: Auto-pruning Policies
authors:
  - "@bcaton85"
reviewers:
  - "@dmage"
  - "@Sunandadadi"
approvers:
  - "@dmage"
  - "@Sunandadadi"
creation-date: 2023-08-15
last-updated: 2023-08-15
status: provisional
---

# Auto-pruning Policies

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. Should there be a dry run feature to test the policy?
1. Should this only be implemented in the new UI and not the old UI?
1. Can a user configure multiple policies per repository?
1. Can a user specify tags or repositories by pattern?

## Summary

In order for a container registry to run cost effectively unused resources should be deleted to reclaim storage space. This task should be done automatically in Quay by deleting unused tags through a given policy. The policy used should be configured by the user and a variety policy options should be given. Examples of initial policies would be prune tags older than X number of days or prune tags when more than 100 are present.

## Motivation

Reclaiming space is essential in running a cost efficient registry. Having this done automatically within Quay removes the need for manual intervention or custom software.

### Goals

* User is able to provide a time based policy for auto-pruning tags
* User is able to provide a tag number based policy for auto-pruning tags
* User is able to specify whether tags go into the time machine or are immediately deleted
* The correct tags are deleted when the policy is applied
* Pruned tags are tracked via usage logs

### Non-Goals

* Allowing users to specify multiple policies per repository
* Applying policies based on repo or tag name pattern matching

## Proposal

This proposal is structured in three parts: how policies are managed, how the policies are structured in the DB, and how the policies will be executed via the auto-pruning worker.

### Managing Policies

A user is able to configure auto-pruning within the repository settings. To start, there will be two auto-pruning options:
* Prune by number of tags, provided the number of tags
* Prune tags older than date, provided the number of days 

For this implementation users will only be able to configure one policy per repository. To interface with this new UI there will a new set of CRUD endpoints for managing auto-pruning policies.

**Create policy**
```
POST /api/v1/repository/{organization}/{repository}/autoprunepolicy
By date:
{
    pruneBy: "creation_date",
    olderThan: "2w"  
}
By number of tags:
{
    pruneBy: "number_of_tags",
    numTags: 100  
}
```

**Update policy**
```
PATCH /api/v1/organization/{organization}/{repository}/autoprunepolicy/{policyid}
{
    fieldToBeUpdated: value,
}
```

**List policies**
```
GET /api/v1/organization/{organization}/{repository}/autoprunepolicy
```

**Get policy**
```
GET /api/v1/organization/{organization}/{repository}/autoprunepolicy/{policyid}
```

**Delete policy**
```
DELETE /api/v1/organization/{organization}/{repository}/autoprunepolicy/{policyid}
```

### Database structure

A table called `autoprunepolicy` will be used to store the repository auto-prune policies.
| Field | Datatype | Attributes | Description|
| --- | ----------- | ----------- | ----------- |
| uuid | character varying(255) | Unique, not null, indexed | Unique identifier for this policy |
| repository_id | integer | FK, not null | Repository that the policy is applied to |
| config | text | JSON, not null | Policy configuration |
| last_ran_ms | bigint | nullable, indexed | Last time this policy was executed |
| is_running | boolean | default false | Policy is currently being executed |

### Auto-pruning worker

The auto-pruning worker is an asyncronous job that executes the policy and performs the following steps:
* Worker starts every 30s or to some configured number
* Select a policy with the least `last_ran_ms` and `is_running`=`false`
    * A policy with a null `last_ran_ms` will also be selected first as this indicates it was never ran. This prioritizes policies that haven't been ran recently.
* Set the policy to `is_running` = `true`
* Determine which pruning method to use based on `config.pruneBy`
* Execute the pruning method (either by number of tags or by date)
    * For pruning by number of tags the worker will get the number of currently active tags sorted by creation date and delete the oldest tags to the configured number
    * For pruning by date the worker will get the active tags older than the specified time span and any tags returned will be deleted
* Add audit logs of the tags deleted
* Set the policy to `is_running` = `false` and update `last_ran_ms` to the current time
* Worker ends

### Constraints

* The speed at which the background worker can execute the retention policies for large registries

### Risks and Mitigations

* Tags are deleted outside of the given policy

### Test Plan

* TBD

### Implementation History

* 2023-08-15 Initial proposal
