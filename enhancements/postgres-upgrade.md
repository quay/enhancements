---
title: Quay Postgres Upgrade
authors:
  - "@jonathankingfc"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2023-01-27
last-updated: 2023-01-27
status: provisional
---

# Quay Operator Postgres Upgrade

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Summary

This enhancement proposes upgrading the version of Postgres on the Red Hat Kubernetes Operator
from version 10 to version 14. Due to the EOL of Postgres 10, the upgrade is critical.
It will also increase performance and provide more features for users.

## Motivation

When Postgre reaches EOL, it will no longer be supported by the devloper community and there
will be no further security updates. Therefore, it is critical to provide an upgrade path
in order to ensure the latest security patches and bug fixes. Postgres 14 also includes a number
of new features which can be used in future versions of Quay.

## Open Questions

What is the best way to infer the database version from the image tag?

### Goals

Smooth Upgrade: The upgrade should happen as a Kubernetes job behind the scenes, and should
maintain
compatibility with all Quay features.

Backups: It is important to take a full backup of the database to ensure that you can revert to the
previous version if needed.

### Non-Goals

Database Backup and Restore (as a feature) / This should be a separate enhancement

## Proposal

### User Stories

#### Ideally, this upgrade happens without any effect on the user. It should happen in the background and should require no manual steps.

#### Users may want to upgrade from < 3.8, they should follow the direct upgrade path.

### Implementation Details/Notes/Constraints

There are two ways to run the `pg_upgrade` command, Linked and In-Place.

In an in-place upgrade, the upgrade is performed directly on the existing data directory of the
older version of postgres. It does not rquire additional resources to be created.

In a linked upgrade, a new object of the target version of postgres is created and the data is
transferred from the old object to the new one. This type of upgrade provides more
safety as the original data remains intact, making it easier to recover in case of an issue.

The linked upgrade will allow us to create a new PVC, copy the users data, and keep the old PVC
intact.

The implementation will happen through a Kubernetes job triggered conditionally in the reconcile loop.
The process would look something like the following:

1. A change to the Quay CR is made, in this case an upgrade, causing the reconcile loop to fire
   off.
2. The reconciler checks the postgres image tag, and determines the current postgres version being
   used.
3. If the postgres version is inspected from the tag and if an upgrade is determined necessary,
   the following events will be triggered.

4. A Kubernetes Job object definition is created include the commands to run a postgres backup before the postgres upgrade.
5. An additional PVC in which the data will be backed up.
6. Retrigger reconcile loop
7. A Kubernetes Job object definition is created include the commands to run a postgres upgrade.
8. Retrigger reconcile loop
   - Check status of upgrade, if upgrade still in process, retrigger every 60 seconds
   - If status of upgrade is complete, let the reconcile loop pass on an do not retrigger

### Risks and Mitigations

The largest risk involved is that of data loss. In order to mitigate this, the backup job must be
completed in order for the upgrade process to happen. After the upgrade, the Operator will point
to the NEW PVC, meaning the clients original data is never touched. This old PVC can be left on
the system, and either manually deleted or cleaned up in a future upgrade.

## Design Details

### Test Plan

Testing can be done using the kuttl framework implemented in the Operator repo. This can be
used to unit test the kubernetes jobs.
Manual upgrades will be done, but for full end to end test on the entire matrix of upgrades, clear
passing requirements should be provided to QE.

### Upgrade / Downgrade Strategy

This is an open question to be discuessed.
Should downgrades be supported? (this is separate from a revert due to failure)

## Drawbacks

There are no clear drawbacks, only risks. This is a required upgrade in order for security to be
maintained.

## Alternatives

There are no alternatives. For the client, the alternative would be to use unmanaged storage with
an upgraded version of Postgres, which is a bad user experience.

## Infrastructure Needed [optional]

There is no additional infrastructure needed.
