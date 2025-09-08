---
title: Last pull metrics
authors:
  - "@harish"
  - "@shubhra"
reviewers:
  - "@bcaton85"
  -
approvers:
  - "@bcaton85"
  - 
creation-date: 2025-09-08
last-updated: 2025-09-08
status: implementable
---

# Last timestamp metrics

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. Will there be pruning policies defined on manifest level metrics?
2. How are we backfilling existing tags and manifest since we don't have historical data?
3. What is the acceptable processing delay for pull statistics updates?
4. Should worker processing frequency be configurable per deployment size? 

## Summary

Currently, users have little to no insight into when an image tag was last pulled. This information is needed to decide whether it is safe to delete a tag as part of cleaning up the repository. We currently record pull events in the audit logs, but these are expensive to query and sometimes do not allow querying at all.

## Motivation

Provide insight into popularity of an image tag by recording the last pull date/ last pull count so that auto-pruning rules can be created that prune based on the least recently pulled images.

### Goals

* User is able to see the last pull timestamp when pulling an image via the tag
* User is able to see the pull count when pulling an image via the tag 
* For each individual image tag, Quay also maintains a record of the last pull timestamp of the associated manifest and updates it on pull via tag or pull via digest
* For each individual image tag, Quay also maintains a pull count the associated manifest and increments it on pull via tag or pull via digest
* When an image is pulled by digest, only manifest digest level last pull metrics are updated
* Last pull metrics for both tag level and manifest level are available via API
* New UI displays the most recent pull date and overall pull count for each image tag. The pull count will be displayed as ranges like 7, 1.7K, 10K+, 50K+, 100K+, 500K+, or 1M+ where as database stores the accurate pull count
* The feature will be gated behind a feature flag
* There is an acceptable delay in showing the accurate last pull events as it is done asynchronously in the background

### Non-Goals

* Real-time pull statistics updates (async processing is acceptable)
* Displaying manifest digest level last pull metrics on UI
* Showing repo-level pull count/ popularity metrics 
* Modifying existing pull endpoints for statistics collection


## Proposal

The proposal details two approaches:
1. Using existing Quay's log model
2. Using redis 

## Design Approach 1: Using existing quay's log model to track last pull events

This approach implements pull metrics tracking using a background worker approach that processes existing audit logs to create aggregated statistics.


```
Pull Events → Audit Logs (existing) → Background Worker → New PullStatistics Table
                  ↓                        ↓                     
            Database/ES logs        Periodic aggregation      
```

### Implementation Approach

**Background Worker Processing**: Instead of modifying high-traffic pull endpoints, a background worker periodically processes existing audit logs to extract pull statistics and store them in an optimized table for fast queries.

**Benefits**:
- Zero impact on pull request performance 
- Works around expensive/inaccessible logs issue
- Handles historical data by processing old logs
- Can recover gracefully from logs access issues

### Database Schema

**TagPullStatistics Table** (Individual tag tracking)

| Field | Type | Description |
|-------|------|-------------|
| repository_id | FK | Repository reference |
| tag_name | varchar | Tag name (e.g., "v1.0", "latest") |
| tag_pull_count | bigint | Count of pulls for this specific tag |
| last_tag_pull_date | datetime | When this specific tag was last pulled |
| current_manifest_digest | varchar | Current manifest this tag points to |

**ManifestPullStatistics Table** (Manifest-level tracking)

| Field | Type | Description |
|-------|------|-------------|
| repository_id | FK | Repository reference |
| manifest_digest | varchar | Manifest digest (unique per repository) |
| total_pull_count | bigint | Total pulls (all tags + digest pulls) |
| last_pull_date | datetime | When manifest was last accessed (any method) |
| last_tag_pulled | varchar | Most recent tag name used to pull this manifest |

**Key Constraints**:
- TagPullStatistics: Unique index on `(repository_id, tag_name)`
- ManifestPullStatistics: Unique index on `(repository_id, manifest_digest)`
- Index on `(repository_id, last_tag_pull_date)` for tag-based pruning queries
- Index on `(repository_id, last_pull_date)` for manifest-based pruning queries

### Background Worker

**PullStatsSyncWorker**: A background service that periodically processes audit logs to maintain pull statistics.

**Processing Logic**:
1. **Query Audit Logs**: Retrieves `pull_repo` log entries since last processing timestamp
2. **Extract Pull Type**: Analyzes log metadata to determine pull method:
   - **Tag pulls**: Log metadata contains `"tag": "v1.0"` field
   - **Digest pulls**: Log metadata contains `"manifest_digest": "sha256:abc..."` field
3. **Update Statistics**:
   - **Tag pulls**: Update both `TagPullStatistics` (specific tag) and `ManifestPullStatistics` (underlying manifest)
   - **Digest pulls**: Update only `ManifestPullStatistics` (no specific tag affected)
4. **Track Processing State**: Maintains last processed timestamp and log ID in `PullStatisticsProcessingState` table to enable incremental updates and avoid reprocessing the same logs
5. **Handle Failures**: Falls back to direct database queries when logs model is inaccessible

**Configuration**:
- Poll interval: 5 minutes (configurable)
- Batch processing: 1000 log entries
- Handles logs model access failures with database fallback

### API Endpoints

**Repository Pull Statistics**
```
GET /api/v1/repository/{repository}/pull_statistics
Response: List of all tag statistics and manifest statistics for the repository
```

**Tag Pull Statistics**  
```
GET /api/v1/repository/{repository}/tag/{tagname}/pull_statistics
Response: {
  "tag_name": "v1.0",
  "tag_pull_count": 150,
  "last_tag_pull_date": "2024-09-08T10:15:00Z",
  "manifest_digest": "sha256:abc123...",
  "manifest_total_pull_count": 500,
  "manifest_last_pull_date": "2024-09-08T10:20:00Z"
}
```

**Manifest Pull Statistics**
```
GET /api/v1/repository/{repository}/manifest/{digest}/pull_statistics  
Response: Manifest-level statistics including all tags and digest pulls
```

### Feature Flag

The feature is gated behind `FEATURE_PULL_STATISTICS_TRACKING` to allow safe rollout and disable if needed.

### Constraints

* **TagPullStatistics** size grows with number of active tags across all repositories
* **ManifestPullStatistics** size grows with unique manifests per repository  
* Processing delay: Statistics updated every 5 minutes, not real-time
* Dependency on audit logs availability (mitigated by database fallback)
* Dual table updates required for tag pulls (slight processing overhead)

### Risks and Mitigations

* **Logs access failures**: Worker includes fallback to query database directly
* **Processing lag**: Configurable polling interval, can be tuned based on load
* **Data consistency**: Worker tracks processing state to avoid duplicate processing

### Test Plan

* Unit tests for worker log processing logic
* Integration tests for API endpoints
* Performance testing with high-volume pull workloads
* Validation of tag vs manifest pull count accuracy

### Implementation History

* 2024-09-08 Initial proposal
