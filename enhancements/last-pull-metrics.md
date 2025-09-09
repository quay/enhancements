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

This enhancement proposes two different approaches for implementing last pull metrics tracking, each with distinct trade-offs:

1. **Approach 1 - Audit Log Processing**: Leverages existing Quay audit logs with background workers for zero pull request impact
2. **Approach 2 - Redis-backed Tracking**: Uses Redis for real-time event capture with periodic database synchronization for better data freshness

Both approaches provide the same core functionality but differ in implementation complexity, infrastructure requirements, and data freshness guarantees. 

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

**Configuration**:
- Poll interval: 5 minutes (configurable)
- Batch processing: 1000 log entries

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

* **Increased Query Volume**
  - Current State: ES queries are primarily for audit/compliance (infrequent)
  - With Pull Stats: Every UI page load, API call, and auto-pruning check queries ES
  - Impact: 100x-1000x increase in query volume
  - Need more ES nodes to handle query load, leads to read-heavy pattern
* **Processing lag**: Configurable polling interval, can be tuned based on load
* **Data consistency**: Worker tracks processing state to avoid duplicate processing

### Test Plan

* Unit tests for worker log processing logic
* Integration tests for API endpoints
* Performance testing with high-volume pull workloads
* Validation of tag vs manifest pull count accuracy

## Design Approach 2: Redis-backed Pull Events Tracking

This approach implements real-time pull metrics tracking by capturing pull events directly in Redis and asynchronously flushing to a persistent database. This provides better performance and lower latency for pull statistics updates.

```
Pull Events → Redis Cache → Background Worker → Persistent Database
     ↓             ↓              ↓                    ↓
Real-time      Temporary        Periodic          Long-term
capture        storage          aggregation       storage
```

### Implementation Approach

**Real-time Event Capture**: Pull events are immediately captured in Redis during image fetch operations (both by tag and by digest), providing near real-time tracking without impacting pull performance.

**Benefits**:
- Near real-time pull statistics updates
- High performance with minimal impact on pull requests
- Resilient to temporary database unavailability
- Natural batching and deduplication through Redis structures
- Lower database load through periodic bulk operations

### Redis Data Structure

**Pull Event Keys**: Each pull event is stored in Redis with a unique key structure that allows for efficient querying and aggregation.

**Key Format**:
```
pull_events:repo:{repository_id}:tag:{tag_name}:{manifest_digest}
pull_events:repo:{repository_id}:digest:{manifest_digest}
```

**Redis Data Structure** (Hash for each key):
```json
{
  "repository_id": "123",
  "tag_name": "v1.0",           // Only for tag pulls
  "manifest_digest": "sha256:abc123...",
  "pull_count": "5",            // Incremented on each pull
  "last_pull_timestamp": "1694168400",
  "pull_method": "tag|digest"   // How the image was pulled
}
```

**Redis Operations**:
- **Tag Pull**: Creates/updates key `pull_events:repo:{repo_id}:tag:{tag}:{digest}`
- **Digest Pull**: Creates/updates key `pull_events:repo:{repo_id}:digest:{digest}`
- **Atomic Updates**: Uses Redis HINCRBY and HSET for thread-safe increments
    - HINCRBY: increments the numeric value of a hash field by the specified amount
    - HSET: Sets a field-value pair in the hash

### Background Worker: RedisFlushWorker

**RedisFlushWorker**: A background service that periodically processes Redis pull events and flushes them to the persistent database.

**Processing Logic**:
1. **Scan Redis Keys**: Uses Redis SCAN to iterate through all `pull_events:*` keys
2. **Batch Processing**: Processes keys in batches of 1000 to avoid memory issues
3. **Aggregate Statistics**: For each repository, aggregates pull counts and timestamps
4. **Database Update**: Performs bulk upsert operations with dual update logic:
   - **Tag pulls**: Update both TagPullStatistics (tag-specific) AND ManifestPullStatistics (manifest-level)
   - **Digest pulls**: Update only ManifestPullStatistics (no tag involved)
5. **Cleanup**: Removes processed keys from Redis after successful database commit
6. **Error Handling**: Retains Redis keys on database failures for retry

**Configuration**:
- Flush interval: 5 minutes (configurable via `REDIS_FLUSH_INTERVAL_SECONDS`)
- Batch size: 1000 keys per processing cycle
- Redis key TTL: 1 day (backup cleanup if worker fails)
- Retry attempts: 3 with exponential backoff

**Processing Flow**:
```python
def process_redis_events():
    for batch in scan_redis_keys("pull_events:*", batch_size=1000):
        tag_updates, manifest_updates = aggregate_pull_events(batch)
        try:
            # Tag pulls update both tables, digest pulls update only manifest table
            if tag_updates:
                bulk_upsert_tag_statistics(tag_updates)
            if manifest_updates:
                bulk_upsert_manifest_statistics(manifest_updates)
            cleanup_redis_keys(batch)
        except DatabaseError:
            # Keys remain in Redis for next cycle
            log_error_and_continue()

def aggregate_pull_events(redis_batch):
    tag_updates = []
    manifest_updates = []
    
    for key, data in redis_batch:
        # Always update manifest stats (both tag and digest pulls)
        manifest_updates.append({
            'repository_id': data['repository_id'],
            'manifest_digest': data['manifest_digest'],
            'pull_count': data['pull_count'],
            'last_pull': data['last_pull_timestamp']
        })
        
        # Additionally update tag stats for tag pulls
        if data['pull_method'] == 'tag':
            tag_updates.append({
                'repository_id': data['repository_id'],
                'tag_name': data['tag_name'],
                'manifest_digest': data['manifest_digest'],
                'pull_count': data['pull_count'],
                'last_pull': data['last_pull_timestamp']
            })
    
    return tag_updates, manifest_updates

def bulk_upsert_tag_statistics(tag_updates):
    """Bulk upsert tag statistics ON CONFLICT"""
    if not tag_updates:
        return
        
    values = []
    for update in tag_updates:
        values.append(f"({update['repository_id']}, '{update['tag_name']}', "
                     f"'{update['manifest_digest']}', {update['pull_count']}, "
                     f"'{update['last_pull']}', NOW())")
    
    sql = f"""
    INSERT INTO TagPullStatistics 
        (repository_id, tag_name, current_manifest_digest, tag_pull_count, 
         last_tag_pull_date, redis_last_processed)
    VALUES {','.join(values)}
    ON CONFLICT (repository_id, tag_name) 
    DO UPDATE SET
        current_manifest_digest = EXCLUDED.current_manifest_digest,
        tag_pull_count = TagPullStatistics.tag_pull_count + EXCLUDED.tag_pull_count,
        last_tag_pull_date = GREATEST(TagPullStatistics.last_tag_pull_date, EXCLUDED.last_tag_pull_date),
        redis_last_processed = EXCLUDED.redis_last_processed
    """
    execute(sql)

**Example: Processing Multiple Tag+Repo+Digest Combinations**

During a 5-minute Redis flush cycle, the worker processes different combinations:

```python
# Input array to bulk_upsert_tag_statistics()
tag_updates = [
    {
        'repository_id': 123,
        'tag_name': 'latest', 
        'manifest_digest': 'sha256abc',
        'pull_count': 15,
        'last_pull': '1694168400'
    },
    {
        'repository_id': 123,
        'tag_name': 'v1.0',
        'manifest_digest': 'sha256abc', 
        'pull_count': 8,
        'last_pull': '1694168350'
    },
    {
        'repository_id': 456,
        'tag_name': 'v2.0',
        'manifest_digest': 'sha256def',
        'pull_count': 3,
        'last_pull': '1694168200'
    },
    {
        'repository_id': 789,
        'tag_name': 'latest',
        'manifest_digest': 'sha256ghi',
        'pull_count': 22,
        'last_pull': '1694168450'
    }
]

# Results in single SQL query processing all combinations:
INSERT INTO TagPullStatistics (...) VALUES 
    (123, 'latest', 'sha256abc', 15, '2024-09-08 10:15:00', NOW()),
    (123, 'v1.0', 'sha256abc', 8, '2024-09-08 10:14:10', NOW()),
    (456, 'v2.0', 'sha256def', 3, '2024-09-08 10:11:40', NOW()),
    (789, 'latest', 'sha256ghi', 22, '2024-09-08 10:15:50', NOW())
ON CONFLICT (repository_id, tag_name) DO UPDATE SET ...
```

### Bulk Upsert Performance Benefits

**Performance Comparison:**
```
Individual Operations:
- 1000 Redis keys = 1000 database queries
- Time: ~5-10 seconds (network overhead)
- Database load: High (connection pool exhaustion)

Bulk Operations:  
- 1000 Redis keys = 2 database queries (tags + manifests)
- Time: ~100-200ms
- Database load: Low (minimal connections)
```

### Database Schema Enhancements

The Redis approach leverages the same database tables as Approach 1 but with optimized bulk update patterns:

**Enhanced TagPullStatistics Table**:
- Adds `redis_last_processed` timestamp for tracking Redis sync state
- Uses bulk UPSERT operations for efficient batch updates
- Updated only when image is pulled by tag name

**Enhanced ManifestPullStatistics Table**:
- Adds `redis_last_processed` timestamp
- Optimized for high-frequency bulk updates
- Updated for ALL pull operations (both tag pulls and direct digest pulls)

**Update Logic**:
- **Tag Pull** (`docker pull repo:tag`): Updates BOTH TagPullStatistics (for the tag) AND ManifestPullStatistics (for the underlying manifest)
- **Digest Pull** (`docker pull repo@sha256:abc...`): Updates ONLY ManifestPullStatistics (no specific tag involved)

This ensures that manifest-level statistics accurately reflect the total usage of the underlying image content, regardless of how it was accessed.

**Required Schema Additions for Redis Approach:**

```sql
-- Add to existing TagPullStatistics table
ALTER TABLE TagPullStatistics 
ADD COLUMN redis_last_processed DATETIME;

-- Add to existing ManifestPullStatistics table  
ALTER TABLE ManifestPullStatistics 
ADD COLUMN redis_last_processed DATETIME;

-- New table for tracking Redis processing state
CREATE TABLE RedisProcessingState (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    last_processed_timestamp DATETIME NOT NULL,
    processing_status ENUM('IDLE', 'PROCESSING', 'ERROR') DEFAULT 'IDLE',
    error_message TEXT,
    keys_processed_count BIGINT DEFAULT 0,
    created_at DATETIME DEFAULT NOW(),
    updated_at DATETIME DEFAULT NOW() ON UPDATE NOW()
);
```

### Integration with Pull Endpoints

**Minimal Code Changes**: Pull endpoints are enhanced with lightweight Redis operations:

```python
def record_pull_event(repository_id, tag_name=None, manifest_digest=None):
    if not feature_flag_enabled("FEATURE_REDIS_PULL_TRACKING"):
        return
        
    if tag_name:
        key = f"pull_events:repo:{repository_id}:tag:{tag_name}:{manifest_digest}"
        pull_method = "tag"
    else:
        key = f"pull_events:repo:{repository_id}:digest:{manifest_digest}"
        pull_method = "digest"
    
    redis_client.hmset(key, {
        "repository_id": repository_id,
        "tag_name": tag_name or "",
        "manifest_digest": manifest_digest,
        "pull_count": 1,
        "last_pull_timestamp": int(time.time()),
        "pull_method": pull_method
    })
    redis_client.hincrby(key, "pull_count", 1)
    redis_client.expire(key, 3600)  # 1 hour TTL as backup
```

### API Endpoints

Uses the same API endpoints as Approach 1 but with faster data availability:

- Statistics are updated within 5 minutes of pull events
- Real-time counters available via Redis for immediate queries (optional endpoint)
- Fallback to database for historical data when Redis keys expire

### Monitoring and Observability

**Metrics to Track**:
- Redis key count and memory usage
- Worker processing latency and throughput
- Database bulk operation performance
- Failed Redis-to-database sync events

**Health Checks**:
- Redis connectivity and memory usage
- Worker processing delays
- Database sync lag monitoring

### Constraints and Considerations

**Redis Memory Usage**:
- Estimated 200 bytes per unique pull event
- For 1M repositories with 10 tags each: ~2GB Redis memory
- TTL-based cleanup prevents unbounded growth

**Processing Guarantees**:
- At-least-once delivery (Redis keys may be reprocessed on worker failures)
- Eventually consistent statistics (5-minute delay maximum)
- Duplicate pull event protection through Redis key structure

**Performance Impact**:
- Minimal database load through batching
- Worker processing scales with Redis memory, not pull volume

### Failure Scenarios and Recovery

**Redis Unavailable**:
- Pull requests continue without statistics tracking
- Worker queues database operations for catch-up
- Feature flag allows disabling Redis integration

**Database Unavailable**:
- Redis continues collecting events
- Worker retries with exponential backoff
- Extended Redis TTL prevents data loss

**Worker Failures**:
- Redis keys persist with TTL for recovery
- Processing state table tracks last successful sync
- Automatic restart picks up from last known state

### Comparison with Approach 1

| Aspect | Approach 1 (Audit Logs) | Approach 2 (Redis) |
|--------|--------------------------|---------------------|
| Data Freshness | 5-minute delay | Near real-time |
| Pull Request Impact | Zero | Minimal (~1-2ms) |
| Infrastructure | Existing logs only | Redis + Database |
| Memory Usage | Low | Moderate (Redis) |
| Failure Recovery | Log reprocessing | Redis persistence |
| Scalability | Limited by log volume | High (Redis scales) |
| Historical Data | Full audit trail | Recent events only |
  - Latency: 10-50ms per query on elastic search vs <1ms for Redis

### Test Plan

**Unit Tests**:
- Redis operations for tag and digest pulls
- Worker batch processing logic
- Database bulk update operations
- Error handling and retry mechanisms

**Integration Tests**:
- End-to-end pull event tracking
- Redis-to-database synchronization
- API endpoint accuracy with Redis data
- Failure scenario recovery

**Performance Tests**:
- High-volume pull request impact
- Redis memory usage under load
- Worker processing throughput
- Database bulk operation efficiency

**Load Testing**:
- 10,000+ concurrent pulls with Redis tracking
- Redis memory usage with 1M+ active keys
- Worker performance with large Redis datasets

### Implementation History

* 2024-09-08 Initial proposal
