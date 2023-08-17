---
title: Namespace Auto-pruning Policies
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

# Namespace Auto-pruning Policies

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. Should there be a dry run feature to test the policy?
1. Should this only be implemented in the new UI and not the old UI?
1. Can a user configure multiple policies per namespace?
1. Can a user specify tags or repositories by pattern?

## Summary

In order for a container registry to run cost effectively unused resources should be deleted to reclaim storage space. This task should be done automatically in Quay by deleting unused tags through a given policy. The policy used should be configured by the user and a variety policy options should be given. Examples of initial policies would be prune tags older than X number of days or prune tags when more than 100 are present.

## Motivation

Reclaiming space is essential in running a cost efficient registry. Having this done automatically within Quay removes the need for manual intervention or custom software.

### Goals

* User is able to provide a time based policy for auto-pruning tags within a namespace
* User is able to provide a tag number based policy for auto-pruning tags within a namespace
* User is able to specify whether tags go into the time machine or are immediately deleted
* The correct tags are deleted when the policy is applied
* Pruned tags are tracked via usage logs

### Non-Goals

* Allowing users to specify multiple policies per namespace
* Applying policies based on repo or tag name pattern matching
* Allowing policy configurations at the repository level

## Proposal

The details in this proposal is only for namespace auto-pruning policies but designed in a way to support repository auto-pruning policies in the future. This proposal is structured in three parts: how policies are managed, how the policies are structured in the DB, and how the policies will be executed via the auto-prune worker.

### Managing Policies

A user is able to configure auto-pruning within the namespace settings. To start, there will be two auto-pruning options:
* Prune by number of tags, provided the number of tags
* Prune tags older than date, provided the number of days 

For this implementation users will only be able to configure one policy per namespace. To interface with this new UI there will be a new set of CRUD endpoints for managing auto-pruning policies. The endpoints returning lists will be paginated.

**Create policy**
```
POST /api/v1/organization/{organization}/autoprunepolicy
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
PATCH /api/v1/organization/{organization}/autoprunepolicy/{policyid}
{
    fieldToBeUpdated: value,
}
```

**List policies**
```
GET /api/v1/organization/{organization}/autoprunepolicy
```

**Get policy**
```
GET /api/v1/organization/{organization}/autoprunepolicy/{policyid}
```

**Delete policy**
```
DELETE /api/v1/organization/{organization}/autoprunepolicy/{policyid}
```

### Database structure
Two tables will be required.

**namespaceautoprunepolicy**

This table holds the policy configuration for a single namespace. For this proposal there will be only one entry per namespace but there will be support for multiple rows per namespace_id. The config field will hold the policy details such as `{pruneBy: "creation_date", olderThan: "2w"}` or `{pruneBy: "number_of_tags",  numTags: 100}`.

| Field | Datatype | Attributes | Description|
| --- | ----------- | ----------- | ----------- |
| uuid | character varying(255) | Unique, not null, indexed | Unique identifier for this policy |
| namespace_id | integer | FK, not null | Namespace that the policy falls under |
| config | text | JSON, not null | Policy configuration |

**autoprunetask**

This table registers tasks to be executed by the auto-prune worker. Tasks are executed within the context of a single namespace. Only one task per namespace will ever exist.

| Field | Datatype | Attributes | Description |
| --- | ----------- | ----------- | ----------- |
| namespace_id | integer | FK, not null | Namespace that this task belongs too |
| last_ran_ms | bigint | nullable, indexed | Last time the worker executed the policies for this namespace |
| is_running | boolean | default false | Policies for this namespace are currently being executed |

While these tables could be combined into a single table they are separated to support a `repositoryautoprunepolicy` in the future.

### Auto-prune worker

**Creation of an auto-prune task**

When ever a policy is created in the `namespaceautoprunepolicy` table a row is also created in the `autoprunetask` table. This will be done within the same transaction. The auto-prune worker will use the entry in the `autoprunetask` table to identify which namespaces it should execute policies for.

**Execution of the auto-prune worker**

The auto-pruning worker is an asyncronous job that executes configured policies. It's workflow is based on values in the `autoprunetask` table.

**Worker Pseudocode**
* Worker starts on a set interval (30s)
* Select a row from `autoprunetask` with the least or null `last_ran_ms` and `is_running`=`false`
    * A null `last_ran_ms` indicates that the task was never ran
    * A task that hasn't been ran in the longest amount of time or has never been ran at all are prioritized.
* Set the task to `is_running` = `true`
* Get the policy configuration from `namespaceautoprunepolicy`
    * If no policy configuration exists, delete the entry from `autoprunetask` for this namespace and exit immediately
* Begin a paginated loop of all repositories under the organization
    * Determine which pruning method to use based on `config.pruneBy`
    * Execute the pruning method with the policy configuration retrieved earlier
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
