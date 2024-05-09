---
title: Trigger Notification on image expiry
authors:
  - "@Sunandadadi"
reviewers:
  - "@kleesc"
  - "@bcaton"
approvers:
  - "@kleesc"
  - "@bcaton"
creation-date: 2024-05-06
last-updated: 2024-05-06
status: implementable
---

# Trigger Notification on image expiry

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

## Summary

Users want to be able to know when an image is about expire, so that do not run into unexpected pull failures.
This feature generates an event that can notify user when an image expires.

### Goals

* Allow users to set up notifications for images expiring in x days (any between 1-30).
* Support notifications via email, Slack webhooks, and etc
* These notifications can be configured at the repository level
* A single notification containing a list of all tags for the repository is to be sent
* All types of image expiration should be captured (including org-level/repo-level auto-prune policies)

## Design Details

**Create Repository Notification**

### API to create new notification

```
POST /api/v1/repository/<apirepopath:repository>/notification/
{
  "config": {<notification menthod config>},
  "event": "repo_image_expiry",
  "method": "slack",
  "title": "Image(s) in test repository will expire in 5 days",
  "eventConfig": {
    "days" : "5" (supported values: 1 to 30),
  },
  "last_ran_ms": null # last time the worker executed this event
}
```

### Database changes

Add a new entry to the `ExternalNotificationEvent` table where name = `repo_image_expiry`. This event is used to
identify `RepositoryNotification` entries that will trigger notifications on image expiry of a repository.

A new table will be added to track tags that were successfully notified.

**tagNotificationSent**

A new entry is added to this table when a notification was sent for a tag. In the future runs of the imageexpiry worker,
this table is used to filter remaining tags for a repository that need to be notified.

| Field | Datatype | Attributes | Description|
| --- | ----------- | ----------- | ----------- |
| id | integer | Unique, not null, auto increment | Unique identifier for the table |
| notification_id | integer | FK, not null | RepositoryNotification.id |
| tag_id | integer | FK, not null | Tag.id |
| method_id | inter | FK, not null | RepositoryNotification.method |

**RepositoryNotification**

Add a new column - `last_ran_ms` to the `RepositoryNotification` table to track when a notification was last processed by the `NotificationWorker`.

| Field | Datatype | Attributes | Description|
| --- | ----------- | ----------- | ----------- |
| last_ran_ms | bigint | nullable | Last time NotificationWorker processed the row |

### Worker

**Pseudocode**

* GC Worker starts on set interval
* Select a row from `RepositoryNotification` where `event = "repo_image_expiry" & number_of_failures < 3` and `FOR UPDATE SKIP LOCKED`
  * null `last_ran_ms` indicates that the task was never ran
  * `number_of_failures` indicates failure by the notificationworker queue  
  * sort query response by `last_ran_ms` asc to prioritize tasks that were never ran or did not run in the longest time
* Get the count of tags that
  * belong to the same repository as the configured notificationEvent, and
  * tag is expiring within configured notificationEvent days
  * include tags expiry due to org-level/repo-level auto-prune policies
  * tag does not belong to `tagnotificationsent` table for the notification method
* If count of tags > 0:
  * push event into notificationworker queue
* Else:
  * Update `event.last_ran_ms` to the current time
* worker ends

This will run as a part of the gc worker to reduce creating more workers.

### NotificationWorker queue

Expected queue data: 
```
{
  "notification_uuid": ...,
  "event_data": {},
  "performer_data": {},
}
```

No significant changes in Notification worker queue. The queue will fetch notification using `notification_uuid`.
Notification Event handler for `repo_image_expiry` is to be built to process the event and notify using the configured
notification method.


### Implementation History

* 2024-05-06 Initial proposal

