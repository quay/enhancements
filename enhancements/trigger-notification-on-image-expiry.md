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
    "last_ran_ms": null # last time the worker executed this event
  },
}
```


### Database changes

Add a new entry to the `ExternalNotificationEvent` table where name = `repo_image_expiry`. This event is used to
identify `RepositoryNotification` entries that will trigger notifications on image expiry of a repository.

### Image-expiry worker

**Worker Pseudocode**

* Worker starts on a set interval
* Select a row from `ExternalNotificationEvent` where `event = "repo_image_expiry" & number_of_failures < 3` and `FOR UPDATE SKIP LOCKED`
  * null `eventConfig.last_ran_ms` indicates that the task was never ran
  * `number_of_failures` indicates failure by the notificationworker queue  
  * sort query response by `eventConfig.last_ran_ms` asc to prioritize tasks that were never ran or did not run in the longest time
* Get the count of tags that
  * belong to the same repository as the configured notificationEvent, and
  * tag is expiring within configured notificationEvent days
  * include tags expiry due to org-level/repo-level auto-prune policies
* If count of tags > 0:
  * push event into notificationworker queue
* Else:
  * Update `event.eventConfig.last_ran_ms` to the current time
* worker ends

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

