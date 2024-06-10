---
title: Annotation-Querying
authors:
  - "@mkok"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-06-07
last-updated: yyyy-mm-dd
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
---

# Querying annotations separately in Quay

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

Users should be able to query a manifest's annotations in quay similar to how there 
already exists the ability to find a manifest's labels via an api call.

## Motivation

Currently in quay, manifest annotations are treated as an add-on to manifest labels. 
When a manifest's labels are queried in quay, a union of the labels and the 
annotations are returned. Annotations will become more widely used in the future, and 
as such there will probably be instances where users will need to query annotations separately
instead of labels. There is also some ambiguity on whether an annotation is present in the manifest
or not due to how quay treats them as labels.


### Goals

* User is able to query annotations with an api endpoint and get the correct annotations back
* Annotations that have special keys such as `quay.expires-after=2d` should have the same behavior
as labels.
* UI should differentiate between labels and annotations

### Non-Goals
* Users can remove annotations from manifests
* Users can see annotations when viewing labels

## Proposal

Make use of the jsonb datatype in postgres to store manifests. This will make it easier
to parse the manifest for the annotations so they are more readily available. The `manifestbytes`
column should be changed to a json type since this is the column where annotations are to be found.

There would need to be another method added to the database model to obtain the annotations from this
column once it is converted to a json type. The current way of listing labels can also be updated to 
refer to the manifest json as well.

### Risks and Mitigations

A database migration is required for this change. Since it will be changing a column, there needs to be
checks to make sure that it is safe for the user to apply this upgrade. 

## Design Details

Currently the manifest_bytes column in the manifest table is of type text

*Manifest Table*

|         Column         |          Type          | Collation | Nullable |               Default                | Storage  | Stats target | Description
|------------------------|------------------------|-----------|----------|--------------------------------------|----------|--------------|-------------
| id                     | integer                |           | not null | nextval('manifest_id_seq'::regclass) | plain    |              |
| repository_id          | integer                |           | not null |                                      | plain    |              |
| digest                 | character varying(255) |           | not null |                                      | extended |              |
| media_type_id          | integer                |           | not null |                                      | plain    |              |
| manifest_bytes         | text                   |           | not null |                                      | extended |              |
| config_media_type      | character varying(255) |           |          |                                      | extended |              |
| layers_compressed_size | bigint                 |           |          |                                      | plain    |              |

The type can be changed to JSONB using an alembic revision. JSONB should be used instead
of JSON because JSONB is significantly faster to process. This will reduce overhead when 
retrieving manifest annotations.

A new endpoint will need to be added that allows users to access manifest annotations.
It should be similar to the labels endpoint to avoid confusion.

**Get annotation(s)**
```
GET /api/v1/repository/{repository}/manifest/{manifestref}/annotations
```

**Update annotation**
```
POST /api/v1/repository/{repository}/manifest/{manifestref}/annotations
{
    "key": "foo",
    "value": "bar"
}
```

For labels the user needs to specify a labelid
e.g. `DELETE /api/v1/repository/{repository}/manifest/{manifestref}/labels/{labelid}`. 
This uses a separate manifest label table, so to keep the design as simple as possible there
is not an endpoint for deleting the annotation.

### API Design

- `/api/v1/repository/{repository}/manfiest/{manifest digest}/annotations`
- same permissions for labels endpoint, require a repo read for GET request and repo write
for POST request
- returns json body with "annotations" key which is a list containing all annotations for the manifest

## Implementation History

## Drawbacks

* Changing the datatype to jsonb may effect tests.
* There may be conflicts when using sqlite for storage if it doesn't support jsonb

## Alternatives

There already exists an `annotations` property on the manifest object that returns
manifest annotations from the parsed bytecode. An alternative would to simply use that property
to specifically return manifest annotations with the new endpoint and not make any changes
to the database schema.

