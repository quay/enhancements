---
title: Redis Cache Provider for Quay
authors:
  - "@syed"
reviewers:
  - "@alecmerdler"
approvers:
  - "@syed"
  - "@alecmerdler"
  - "@kleesc"
creation-date: 2021-04-09
last-updated: 2021-04-14
status: implemented
---

# redis-cache-provider 

Quay has a data caching mechanism which it uses to cache various objects.
This proposal is to add Redis as another provider/implementation for this cache.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions

 > 1. What happens if external cache fails? Do we fallback to internal cache? 

## Summary

Quay relies heavily on a Cache to reduce the number of queries going to the DB.
This becomes critical for large deployments like quay.io where DB queries are
very expensive. This proposal is to add a shared cache using Redis as the cache store.

## Motivation

Current implementations of Cache (`InMemoryDataModelCache`,
`MemcachedModelCache`) are local to the Pod. This means

1. Each Pod maintains it's own cache and if a request for a resource goes to
   another pod it will have to query the DB even though it was cached earler by
   another Pod. 

2. Restarting a Pod makes it loose all the cache and if multiple pods restart in a
   very short time, it can put significant load on the DB.

What we need is a central cache which is persistent across Pod restarts. We
currently have memcache which could serve as a shared cache if we spin up a
memcache cluster outside of the Pod and point to it from the config. However,
one of the main drawbacks of Memcache is that it does not support HA.

This leaves us with Redis which supports HA among multiple replicas. Redis also
has the added advantage of storing the cache on the disk which is useful in case
of a restarts where we don't have to warm the cache from scratch. Redis also supports
snapshots and restore for a quick turnaround in case of failure.

### Goals

* Implement a Redis backed cache
* Validate the implemenatation works with HA/Replicated Redis

### Non-Goals

* Exact implementation of Redis (Elasticache Cluster vs Replica vs Sentinel)
* Upgrades/Patches to Redis. This is expected to be handled from the service
  without any significant downtime visible to Quay

## Design Details

Initial work around adding this functionality was done by @alecmerdler in [this PR](https://github.com/quay/quay/pull/444)
However, more work is needed to support HA with AWS and Redis Sentinel.

### Redis High Availability (Elasticache)

Elasticache has two different modes of High availability

#### Redis Cluster Mode

This mode requires use of a client library that is cluster aware. This mode is
useful for large caches where sharding is required to achieve horizontal
scalability. Based on our analysis on traffic on Quay.io we don't see the need
for sharding as our data footprint is very small. Hence, we do not recomended
this method for Quay.
   
#### Redis non-Cluster Mode

In this mode we have multiple Redis instances which share the same data i.e no
sharding. There is a one write master and multiple read-only replicas. The
redis client from the application connects to a `Primary Endpoint` exposed by
Elasticache and AWS manages the failover in case of a crash to another node so
the client can be agnostic to upgrade/changes to redis. This works well for
Quay as it gives us HA without the complexity of sharding.

![](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2021/03/10/Screen-Shot-2021-03-10-at-09.44.22.png)

**NOTE** Other cloud providers (GCP, Azure) offer similar HA mode. At this point in time, GCP doesn't support the Cluster Mode for HA.

### Standalone Redis High Availability (Sentinel)

Standalone Redis doesn't support HA. Redis Sentinel is the solution which adds
things like Monitoring, HA & Failover on top of Redis. Sentinel is deployed on independent
nodes outside of the redis cluster and keeps track of the master/slave nodes. When the master goes
down Sentinel fails over to the slave node and promotes it to be new new master. The client application
speaks to the sentinel node(s) to get the list of master and slave nodes

![](https://miro.medium.com/max/700/1*gszoEBW0lupbMDDGGgYOPA.png)


### Risks and Mitigations

* The biggest risk is if the shared cache crashes or is unreachable. In
  production workloads it is required to have some form of HA for Redis

* Redis, for the most part is a single threaded application. Although the reads
  are pertty quick, for huge traffic this could become a potential bottleneck


### Test Plan

* Deploy Redis in both AWS Elasticache and on an EC2
* Both implementations should be functionally identical

**TODO(alecmerdler)** Add detailed test plan


### Upgrade / Downgrade Strategy

Move to Redis cache should be transparent to the deployment and can be deployed
in a rolling fashion. The new pods will start using the Redis Cache while the old pods
will still use the internal cache and will eventually be replaced with the new pods which
use Redis

**TODO(alecmerdler):** Test plan

### Configuration

the `config.yaml` for this enhancement:

```
DATA_MODEL_CACHE_CONFIG:
  engine: redis
  primary_host: <redis-primary-hostname>
  primary_port: <redis-read-port>
  replica_host: <redis-replica-hostname>
  replica_port: <redis-read-port>
  password: <redis-password>
  ssl: <true or false>
  ca_cert: <cert if using SSL>
```

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

* 2021-04-09 Initial proposal
* 2021-04-14 Redis Cache implementation merged (https://github.com/quay/quay/pull/444)
* 2021-05-20 Added configruation for read and write endpoints ()

## Drawbacks

* Network latency added due to moving to an external cache
* Dependency on another component outside of the application
* If Redis is deployed in a non-HA fashion it can become a single point of failure

## Alternatives

* Use a local memcache or in-memory cache
* Have a faster DB for cases where there is a cache failure

## References

* https://aws.amazon.com/blogs/database/configuring-amazon-elasticache-for-redis-for-higher-availability/
* https://medium.com/@amila922/redis-sentinel-high-availability-everything-you-need-to-know-from-dev-to-prod-complete-guide-deb198e70ea6
