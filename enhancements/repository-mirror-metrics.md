---
title: repository-mirror-metrics
authors:
  - "@jortizpa"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-10-15
last-updated: 2025-12-09
status: provisional
see-also:
  - "https://issues.redhat.com/browse/RFE-6452"
---

# Repository Mirror Metrics and Health Endpoints

This enhancement introduces comprehensive metrics and health endpoints for Quay's repository mirroring functionality. The goal is to provide improved visibility, monitoring, and alerting for mirror worker operations, with data available on a per-mirror basis.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

1. Should the health endpoint be exposed on the same port as existing Quay metrics, or should it have a dedicated endpoint?

## Summary

Currently, Quay's repository mirroring lacks the detailed observability required for robust operations. The sole existing metric, quay_repository_rows_unmirrored, provides insufficient insight for proactive monitoring and alerting.
This enhancement addresses these gaps by introducing four new, specific metrics and a dedicated health endpoint for the mirror workers.
These additions will empower operators to:

- Monitor the real-time health and performance of mirroring operations.
- Configure proactive alerts for synchronization failures or delays.
- Track synchronization progress on a per-repository basis.
- Rapidly diagnose and resolve mirroring issues.
- Make data-driven decisions on resource allocation and capacity planning.

## Motivation

Repository mirroring is a critical feature for many Quay deployments, enabling organizations to maintain synchronized copies of upstream container repositories. However, without proper observability, operators face several challenges:

1. **Lack of visibility**: No way to determine if mirror workers are functioning correctly
2. **Reactive troubleshooting**: Issues are only discovered after users report problems
3. **No alerting capability**: Cannot set up automated alerts for synchronization failures
4. **Limited metrics**: The single `quay_repository_rows_unmirrored` metric doesn't provide enough detail
5. **Operational inefficiency**: Difficult to track which repositories are successfully mirrored and which are failing

This enhancement addresses these gaps by providing comprehensive metrics and health endpoints specifically designed for repository mirroring operations.

### Goals

1. **Provide four new metrics for repository mirroring:**
   - Tags pending synchronization: Total number of tags not yet synchronized for each mirrored repository
   - Status of the last synchronization: An indicator (success/fail/in-progress) of the latest synchronization attempt per repository
   - Complete synchronization per repository: A boolean metric (0/1) indicating if a specific mirrored repository has successfully synchronized all its tags since the last run
   - Synchronization failure counter: A cumulative counter of mirroring failures per repository for alerting purposes

2. **Implement a health endpoint** that provides real-time status information about mirror workers, including:
   - Number of active mirror workers
   - Current operational status of each worker
   - Last successful synchronization timestamp
   - Any error conditions or warnings

3. **Enable Prometheus-based alerting** by exposing metrics in a format compatible with Prometheus/AlertManager

4. **Maintain backward compatibility** with existing monitoring setups and the current `quay_repository_rows_unmirrored` metric

5. **Provide comprehensive documentation** for setting up monitoring dashboards and alerts

### Non-Goals

1. This enhancement does not propose changes to the repository mirroring functionality itself
2. This enhancement does not include automatic remediation or self-healing capabilities for failed synchronizations
3. Historical metric storage and long-term trending are not part of this proposal (this should be handled by existing monitoring infrastructure)
4. This enhancement does not change the existing mirroring configuration or scheduling mechanisms
5. UI/dashboard creation in Quay's web interface is not included (metrics can be consumed by external tools like Grafana)

## Proposal

### User Stories

#### Story 1: Operations Engineer Setting Up Monitoring

As an operations engineer responsible for maintaining a Quay deployment, I want to monitor the health of repository mirroring so that I can ensure that our mirrored repositories are staying up-to-date with their upstream sources.

I need to:
- Query a health endpoint to verify all mirror workers are operational
- Set up Prometheus alerts that fire when synchronization failures exceed a threshold
- Create Grafana dashboards showing synchronization status across all mirrored repositories
- Track the number of pending tags to understand the current workload

#### Story 2: Site Reliability Engineer Troubleshooting Issues

As an SRE investigating why users are reporting outdated images in mirrored repositories, I want detailed metrics about synchronization status so that I can quickly identify which repositories are failing to sync and why.

I need to:
- Check the last synchronization status for each repository
- Identify repositories with incomplete synchronizations
- View the failure counter to see which repositories are consistently failing
- Correlate synchronization issues with other system metrics


### Implementation Details/Notes/Constraints

#### Proposed Metrics

All metrics should be exposed via the existing Quay metrics endpoint (typically `/metrics`) in Prometheus format.

**1. Tags Pending Synchronization**
```
# HELP quay_repository_mirror_tags_pending Total number of tags pending synchronization for each mirrored repository
# TYPE quay_repository_mirror_tags_pending gauge
quay_repository_mirror_tags_pending{namespace="org1",repository="repo1"} 5
quay_repository_mirror_tags_pending{namespace="org2",repository="repo2"} 0
```

**2. Last Synchronization Status**
```
# HELP quay_repository_mirror_last_sync_status Status of the last synchronization attempt
# TYPE quay_repository_mirror_last_sync_status gauge
quay_repository_mirror_last_sync_status{namespace="org1",repository="repo1",status="success",last_error_reason=""} 1
quay_repository_mirror_last_sync_status{namespace="org2",repository="repo2",status="failed",last_error_reason="auth_failed"} 1
quay_repository_mirror_last_sync_status{namespace="org3",repository="repo3",status="in_progress",last_error_reason=""} 1
```

Note: This metric follows the Prometheus State Metric pattern where the value is always 1 when that specific status is active. The `status` label indicates the current state: `success`, `failed`, or `in_progress`. The `last_error_reason` label contains the failure reason when `status="failed"`, mirroring the `reason` label values from `quay_repository_mirror_sync_failures_total`. When status is `success` or `in_progress`, the `last_error_reason` label is empty. This design makes aggregations meaningful - for example, `sum(quay_repository_mirror_last_sync_status{status="failed"})` provides an instant count of all currently failing mirrors across the entire registry.

**3. Complete Synchronization Status**
```
# HELP quay_repository_mirror_sync_complete Indicates if all tags have been successfully synchronized (0=incomplete, 1=complete)
# TYPE quay_repository_mirror_sync_complete gauge
quay_repository_mirror_sync_complete{namespace="org1",repository="repo1"} 1
quay_repository_mirror_sync_complete{namespace="org2",repository="repo2"} 0
```

**4. Synchronization Failure Counter**
```
# HELP quay_repository_mirror_sync_failures_total Total number of synchronization failures per repository
# TYPE quay_repository_mirror_sync_failures_total counter
quay_repository_mirror_sync_failures_total{namespace="org1",repository="repo1",reason="network_timeout"} 3
quay_repository_mirror_sync_failures_total{namespace="org2",repository="repo2",reason="auth_failed"} 7
```

**Additional Supporting Metrics**
```
# HELP quay_repository_mirror_workers_active Number of currently active mirror workers
# TYPE quay_repository_mirror_workers_active gauge
quay_repository_mirror_workers_active 5

# HELP quay_repository_mirror_last_sync_timestamp Unix timestamp of the last synchronization attempt
# TYPE quay_repository_mirror_last_sync_timestamp gauge
quay_repository_mirror_last_sync_timestamp{namespace="org1",repository="repo1"} 1697385600

# HELP quay_repository_mirror_sync_duration_seconds Duration of the last synchronization operation
# TYPE quay_repository_mirror_sync_duration_seconds histogram
quay_repository_mirror_sync_duration_seconds_bucket{namespace="org1",repository="repo1",le="30"} 45
quay_repository_mirror_sync_duration_seconds_bucket{namespace="org1",repository="repo1",le="60"} 82
quay_repository_mirror_sync_duration_seconds_bucket{namespace="org1",repository="repo1",le="120"} 95
quay_repository_mirror_sync_duration_seconds_bucket{namespace="org1",repository="repo1",le="+Inf"} 100
```

#### Health Endpoint

A new HTTP endpoint should be added: `/health/mirror` (or `/api/v1/repository/mirror/health`)

**Response format (JSON):**
```json
{
  "healthy": true,
  "workers": {
    "active": 5,
    "configured": 5,
    "status": "healthy"
  },
  "repositories": {
    "total": 150,
    "syncing": 3,
    "completed": 145,
    "failed": 2
  },
  "tags_pending": 47,
  "last_check": "2025-10-15T10:30:00Z",
  "issues": []
}
```

When issues are present:
```json
{
  "healthy": false,
  "workers": {
    "active": 3,
    "configured": 5,
    "status": "degraded"
  },
  "repositories": {
    "total": 150,
    "syncing": 2,
    "completed": 140,
    "failed": 8
  },
  "tags_pending": 234,
  "last_check": "2025-10-15T10:30:00Z",
  "issues": [
    {
      "severity": "warning",
      "message": "2 mirror workers are not responding",
      "timestamp": "2025-10-15T10:25:00Z"
    },
    {
      "severity": "error",
      "message": "Repository org2/repo2 has failed 10 consecutive synchronization attempts",
      "timestamp": "2025-10-15T10:20:00Z"
    }
  ]
}
```

#### Implementation Approach

1. **Metrics Collection Layer**: Add a new metrics collection module in the mirror worker codebase that tracks:
   - Synchronization attempts and their outcomes
   - Current state of each mirrored repository
   - Worker health and activity status
   - Tags pending synchronization

2. **Data Storage**: Utilize existing database tables or add new fields to track:
   - Last sync timestamp
   - Last sync status
   - Failure count
   - Tags sync status

3. **Metrics Exporter**: Extend the existing Prometheus metrics endpoint to expose the new mirror-specific metrics

4. **Health Endpoint**: Implement a new REST API endpoint that aggregates mirror worker status and provides a quick health check

5. **Documentation**: Provide example Prometheus alert rules and Grafana dashboard configurations

### Risks and Mitigations

**Risk 1: Performance Impact**
- *Risk*: Collecting and exposing detailed per-repository metrics could impact database performance with many mirrored repositories
- *Mitigation*: 
  - Implement efficient database queries with proper indexing
  - Use caching for metrics that don't require real-time accuracy
  - Consider aggregating metrics for very large deployments
  - Provide configuration options to adjust metrics granularity

**Risk 2: Cardinality Explosion**
- *Risk*: With many repositories, the number of metric series could become very large
- *Mitigation*:
  - Implement configurable limits on metric cardinality
  - Provide options to aggregate metrics at the namespace level
  - Document best practices for metric collection intervals
  - Consider using metric relabeling in Prometheus

**Risk 3: Security and Information Disclosure**
- *Risk*: Metrics might expose sensitive information about repository names or synchronization patterns
- *Mitigation*:
  - Ensure metrics endpoint requires authentication
  - Follow existing Quay security patterns for API endpoints
  - Allow configuration to disable detailed per-repository metrics
  - Document security considerations in the deployment guide

**Risk 4: Backward Compatibility**
- *Risk*: Changes to existing metrics might break existing monitoring setups
- *Mitigation*:
  - Keep existing `quay_repository_rows_unmirrored` metric unchanged
  - Add new metrics alongside existing ones
  - Provide migration guide for updating monitoring configurations
  - Version the health endpoint API

## Design Details

### Test Plan

#### Unit Tests
- Test metrics collection functions for each new metric
- Test health endpoint response generation with various worker states
- Test error handling when mirror workers are unavailable
- Test metric label generation for different repository configurations

#### Integration Tests
- Test complete synchronization flow with metrics collection
- Test metrics accuracy during normal operations
- Test metrics during failure scenarios (network issues, auth failures, etc.)
- Test health endpoint with multiple concurrent requests
- Test metrics endpoint performance with large numbers of repositories

#### End-to-End Tests
- Set up a Quay instance with multiple mirrored repositories
- Configure Prometheus to scrape the metrics endpoint
- Trigger various synchronization scenarios (success, failure, in-progress)
- Verify metrics are accurately reported in Prometheus
- Verify health endpoint returns correct status
- Test alert firing based on failure thresholds

### Graduation Criteria

#### Dev Preview
- Core metrics are implemented and exposed via Prometheus endpoint
- Health endpoint provides basic status information
- Unit and integration tests pass

#### Tech Preview
- All four primary metrics are fully implemented
- Health endpoint provides comprehensive status details
- Example Prometheus alert rules are provided
- User-facing documentation is complete
- Feedback 

#### GA
- Metrics have been validated in production environments
- Performance optimization is complete
- All edge cases are handled properly
- Documentation includes troubleshooting guide
- Backward compatibility is maintained
- Security review is complete
- Sufficient customer feedback has been incorporated

### API Design

#### Metrics Endpoint
- **Path**: `/metrics` (existing endpoint, extended with new metrics)
- **Method**: GET
- **Authentication**: Same as existing metrics endpoint (typically requires authentication in production)
- **Response Format**: Prometheus text format

#### Health Endpoint
- **Path**: `/api/v1/repository/mirror/health`
- **Method**: GET
- **Authentication**: Requires authenticated user with appropriate permissions
- **Permissions**: Same as viewing repository information (user must have access to at least one mirrored repository)
- **Response Format**: JSON
- **Response Codes**:
  - 200: Success, returns health status with `"healthy": true`
  - 503: Service Unavailable, returns health status with `"healthy": false` (e.g., zero workers, critical errors)
  - 401: Unauthorized
  - 403: Forbidden (user doesn't have required permissions)
  - 500: Internal server error

Note: The HTTP status code reflects the overall mirror worker health status. When the JSON response body shows `"healthy": true`, the endpoint returns 200 OK. When `"healthy": false` due to any critical issue (e.g., zero active workers, database connectivity issues, or persistent synchronization failures), the endpoint returns 503 Service Unavailable. This allows monitoring tools and load balancers to immediately determine health status from the HTTP status code without parsing the JSON body.

**Optional Query Parameters**:
- `namespace`: Filter health check to specific namespace
- `detailed`: Include per-repository breakdown (default: false)

Example: `/api/v1/repository/mirror/health?namespace=myorg&detailed=true`

### Upgrade / Downgrade Strategy

**Upgrade**:
- New metrics will automatically become available after upgrade
- No configuration changes required for basic functionality
- Existing monitoring setups will continue to work
- Operators should update their Prometheus configuration to scrape new metrics
- Operators should create new dashboards and alerts to take advantage of new capabilities

**Downgrade**:
- New metrics will no longer be available
- Health endpoint will return 404
- Existing `quay_repository_rows_unmirrored` metric will continue to work
- Prometheus will log warnings about missing metrics (these can be ignored)
- Any dashboards or alerts using new metrics will need to be disabled or removed

### Version Skew Strategy

In multi-component Quay deployments:
- Metrics collection is implemented in the mirror worker component
- If mirror workers are running newer versions than the main Quay application, new metrics will be available
- If mirror workers are running older versions, new metrics will not be available (but system remains functional)
- Health endpoint is implemented in the main Quay application
- Mixed version deployments should upgrade mirror workers first, then the main application
- No coordination between components is required; metrics will simply appear as workers are upgraded

## Implementation History

- 2024-10-04: RFE-6452 created by customer
- 2025-10-15: Initial enhancement proposal created

## Drawbacks

While this enhancement provides significant operational benefits, there are some considerations to be aware of:

1. **Modest Resource Overhead**: Collecting and exposing metrics will consume some additional resources. However, this overhead is minimal compared to the operational benefits gained. The implementation will use efficient caching and optimized database queries to minimize impact. For most deployments, the resource consumption will be negligible.

2. **Metric Cardinality Management**: Per-repository metrics could increase Prometheus cardinality in very large deployments. This is addressed through configurable aggregation options and follows established Prometheus best practices. Organizations can choose the appropriate level of granularity for their needs, and we'll provide clear documentation on cardinality management.

3. **Initial Setup Effort**: Operators will need to configure their monitoring systems to collect and visualize the new metrics. However, we will provide ready-to-use Prometheus alert rules and Grafana dashboard examples, significantly reducing the setup time. The investment in setup is quickly offset by the operational efficiency gains.

## Alternatives

### Alternative 1: Enhanced Logging Instead of Metrics

Enhance existing logging with detailed synchronization status and rely on log aggregation tools for monitoring. Rejected because it requires separate log parsing infrastructure, lacks standardization, and is less efficient for querying and alerting compared to Prometheus metrics.

### Alternative 2: External Monitoring Agent

Develop a separate sidecar agent that polls the Quay database and exposes metrics independently. Rejected because it introduces a new component requiring independent maintenance, adds deployment complexity, and creates version compatibility challenges.

### Alternative 3: Webhook-Based Event System

Implement webhooks to push synchronization events to external monitoring systems. Rejected because it requires complex event delivery infrastructure, external state management, and only shows changes rather than current state.

### Rationale for Proposed Solution

The Prometheus metrics and health endpoint approach leverages existing infrastructure already used by Quay, provides a standardized interface, requires minimal additional code, and integrates naturally with Kubernetes/OpenShift monitoring stacks.

## Infrastructure Needed

1. **Update Documentation**: To add user-facing documentation about the new metrics and health endpoint

2. **Example Dashboard**: The documentation should also host and maintain example Grafana dashboards and Prometheus alert rules

3. **Testing Infrastructure**: Test environments with:
   - Multiple mirrored repositories
   - Prometheus and Grafana setup
   - Ability to simulate various failure scenarios

4. **CI/CD Updates**: Integration tests that verify metrics accuracy and health endpoint functionality

## Appendix

### Example Prometheus Alert Rules

```yaml
groups:
  - name: quay_mirror_alerts
    interval: 30s
    rules:
      - alert: QuayMirrorSyncFailures
        expr: rate(quay_repository_mirror_sync_failures_total[5m]) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Repository mirroring failures detected"
          description: "Repository {{ $labels.namespace }}/{{ $labels.repository }} has {{ $value }} failures per second"

      - alert: QuayMirrorHighFailureCount
        expr: quay_repository_mirror_sync_failures_total > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High number of mirror synchronization failures"
          description: "Repository {{ $labels.namespace }}/{{ $labels.repository }} has {{ $value }} total failures"

      - alert: QuayMirrorSyncStale
        expr: time() - quay_repository_mirror_last_sync_timestamp > 3600
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Mirror synchronization is stale"
          description: "Repository {{ $labels.namespace }}/{{ $labels.repository }} hasn't synced in over an hour"

      - alert: QuayMirrorWorkersDown
        expr: quay_repository_mirror_workers_active == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "No active mirror workers"
          description: "All mirror workers are down or not responding"

      - alert: QuayMirrorHighPendingTags
        expr: sum(quay_repository_mirror_tags_pending) > 1000
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High number of pending tags"
          description: "There are {{ $value }} tags pending synchronization across all repositories"
```

### Example Grafana Dashboard Queries

**Panel 1: Synchronization Status Overview**
```promql
sum by(namespace, repository, status) (quay_repository_mirror_last_sync_status)
```

**Panel 2: Count of Failed Mirrors**
```promql
sum(quay_repository_mirror_last_sync_status{status="failed"})
```

**Panel 3: Failure Rate**
```promql
rate(quay_repository_mirror_sync_failures_total[5m])
```

**Panel 4: Pending Tags by Repository**
```promql
topk(10, quay_repository_mirror_tags_pending)
```

**Panel 5: Active Mirror Workers**
```promql
quay_repository_mirror_workers_active
```

**Panel 6: Synchronization Duration**
```promql
histogram_quantile(0.95, rate(quay_repository_mirror_sync_duration_seconds_bucket[5m]))
```

