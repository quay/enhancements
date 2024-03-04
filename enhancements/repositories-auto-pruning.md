---
title: Repository Auto-pruning Policies
authors:
  - "@harishsurf"
reviewers:
  - "@bcaton85"
  - "@dmage"
approvers:
  - "@bcaton85"
  - "@dmage"
creation-date: 2024-15-01
last-updated: 2024-15-01
status: implementable
---

# Repositories Auto-pruning Policies

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

In order for a container registry to run cost effectively unused resources should be deleted to reclaim storage space. This task should be done automatically in Quay by deleting unused tags through a given policy. The policy used should be configured by the user and a variety policy options should be given. Examples of initial policies would be prune tags older than X number of days or prune tags when more than 100 are present.

## Motivation

Reclaiming space is essential in running a cost efficient registry. Having this done automatically within Quay removes the need for manual intervention or custom software.

### Goals

* User is able to provide a time based policy for auto-pruning tags within a repository.
* User is able to provide a tag number based policy for auto-pruning tags within a repository.
* Namespace and repository policy are evaluated and a wider policy is applied.
* Pruned tags are tracked via usage logs.

### Non-Goals

* Allowing users to specify multiple policies per repository
* Applying policies based on tag name pattern matching
* User is able to specify whether tags go into the time machine or are immediately deleted

## Proposal

The details in this proposal is to support repository auto-pruning policies. This proposal is structured in three parts: how policies are managed, how the policies are structured in the DB, and how the policies will be executed via the auto-prune worker.

### Managing Policies

A user is able to configure auto-pruning within the repository settings. To start, there will be two auto-pruning options:
* Prune by number of tags: when the actual number of tags exceed the desired number, the oldest tags will be deleted by creation date until the desired number is achieved
* Prune tags by creation date: any tags with a creation date older than the given timespan (e.g. 14 days) will be deleted

For this implementation users will only be able to configure one policy per repository. To interface with this new UI there will be a new set of CRUD endpoints for managing auto-pruning policies. The endpoints returning lists will be paginated.

**Create policy**
```
POST /api/v1/repository/{apirepopath:repository}/autoprunepolicy
By date:
{
    method: "creation_date",
    olderThan: "2w"  
}
By number of tags:
{
    method: "number_of_tags",
    numTags: 100  
}
```

**Update policy**
```
PATCH /api/v1/repository/{apirepopath:repository}/autoprunepolicy/{policyid}
{
    fieldToBeUpdated: value,
}
```

**List policies**
```
GET /api/v1/repository/{apirepopath:repository}/autoprunepolicy
```

**Get policy**
```
GET /api/v1/repository/{apirepopath:repository}/autoprunepolicy/{policyid}
```

**Delete policy**
```
DELETE /api/v1/repository/{apirepopath:repository}/autoprunepolicy/{policyid}
```

### Database structure
A new table will be created to store repository policies.

**repositoryautoprunepolicy**

This table holds the policy configuration for a single repository. For this proposal there will be only one entry per repository but there will be support for multiple rows per `repository_id` in future. The policy field will hold the policy details such as `{method: "creation_date", olderThan: "2w"}` or `{method: "number_of_tags",  numTags: 100}`. `repository_id` is a foreign key that references the `repositories` table.

| Field | Datatype | Attributes | Description|
| --- | ----------- | ----------- | ----------- |
| uuid | character varying(255) | Unique, not null, indexed | Unique identifier for this policy |
| repository_id | integer | FK, not null | Repository that the policy falls under |
| namespace_id | integer | FK, not null | Namespace that the policy falls under |
| policy | text | JSON, not null | Policy configuration |

**autoprunetaskstatus**

This existing table will continue to be used to execute repository pruning policies. A new entry will only be created if there is no entry for the repo's namespace for a given repository policy. 

### Auto-prune worker

**Creation of an auto-prune task**

Whenever a policy is created in the `repositoryautoprunepolicy` table, there can be two cases: 
* if the `autoprunetaskstatus` doesn't have a row for the namespace the repository belongs to - a new row is created with the repo's namespace as `namespace_id`, `status="queued"` and `last_ran_ms=None` in the `autoprunetask` table. This will be done within the same transaction. 
* if the `autoprunetaskstatus` already has a row for the namespace - no update is done to the `autoprunetaskstatus` table.

The auto-prune worker will use the entry in the `autoprunetaskstatus` table to identify which namespaces it should execute policies for.

**Execution of the auto-prune worker**

The existing auto-pruning worker will be refactored to support both namespace and repository level policies. It's workflow is based on values in the `autoprunetaskstatus` table.

**Worker Pseudocode**

* Worker starts on a set interval (1hr)
* Select a row from `autoprunetaskstatus` with the least or null `last_ran_ms` and `FOR UPDATE SKIP LOCKED`
    * A null `last_ran_ms` indicates that the task was never ran
    * A task that hasn't been ran in the longest amount of time or has never been ran at all are prioritized.
* Get the namespace policy(s) configuration for the namespace from `namespaceautoprunepolicy`
    * If no namespace policy exists, look up repository policy(s) under the given namespace. 
        * If no repository policy exist, delete the entry from `autoprunetaskstatus` for this namespace and exit immediately.
        * If repository policies exist, skip deleting `autoprunetaskstatus` table entry. Instead loop through all repository policies and apply them sequentially.
    * If namespace policy(s) exists, begin a paginated a loop for all repositories under the namespace.
        1. Get the repository policy(s) for the repository
        2. Reconcile and execute the namespace policy(s) and the repository policy(s). Here we enforce rules and apply logical operators on which policies gets enforced between each level. How exactly those policies are reconciled is explained in below: 
            - Policy is selected based on namespace-repository policy matrix shown below:

#### When policy method type are same for both namespace and repository level:   
- The convention is to always pick a wider policy when comparing namespace vs repo policy   

| Namespace policy | Repo policy | Condition | Result|
| -----------------| ----------- | ----------- |----------- |
| # of tags, (x tags)  | # of tags, (y tags) | when (x < y)| If there are more than x tags, delete the oldest tags until x remain. | 
|                      |                     | when (x > y)| If there are more than x tags, delete the oldest tags until y remain. |
| Tag age, (x days)    | Tag age, (y days) | when (x < y) |  Delete any tags older than x days |
|                      |                   | when (x > y) | Delete any tags older than y days  | 

#### When the policy method is different for namespace vs repository level
- Apply each of the policy sequentially. Order doesn't matter as both the policy methods rely on tag creation date to prune older tags.

| Namespace policy | Repo policy | Condition | Result|
| -----------------| ----------- | ----------- |----------- |
| # of tags, (x tags) | Tag age, (y days) |            | If there are more than #x tags, delete the oldest tags until #x remain **followed by** deleting older tags if it is older than y days | 
| Tag age, (x days)    | # of tags, (y tags) |          | If there are tags older than x days, delete tags older than x days **followed by** deleting older tags until #y tags remain |

* End looping for repository policies
* Add audit logs of the tags deleted
* Update `last_ran_ms` to the current time
* Worker ends

### Constraints

* The speed at which the background worker can execute the retention policies for large registries

### Risks and Mitigations

* Tags are deleted outside of the given policy

### Test Plan

* TBD

### Implementation History

* 2024-15-01 Initial proposal