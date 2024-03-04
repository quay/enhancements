---
title:  OMR DB migration
authors:
  - "@harishsurf"
reviewers:
  - "@dmage"
  - "@jonathankingfc"
approvers:
  - "@dmage"
  - "@jonathankingfc"
creation-date: 2023-11-28
last-updated: 2023-11-28
status: 
---

# OMR DB migration

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA

## Open Questions [optional]

1. How much additional space should we tell customers they need for the db backup? Do we want to persist the backup db file post migration?
2. How much downtime can we expect?
3. Do we provide an option to change the storage path for SQLite?

## Summary
Postgres 10 is going EOL inside Red Hat in May 2024. OMR uses Postgres 10 and there is no built in logic for updating. This enhancement drafts a plan to provide SQLite as the default database for fresh OMR installs and also have a migration path for transferring data from an existing postgres 10 to SQLite database.

## Motivation

Reclaiming space is essential in running a cost efficient OMR in a disconnected environment. Switching over to Sqlite database will help reduce the OMR tarball size making it easily shippable. 

### Goals

* Use SQLite database as the default db for fresh OMR installs.
* Define a migration path for transferring data from an existing postgres 10 to SQLite database.

### Non-Goals

* 

## Proposal
The design proposal is split into two parts: One for fresh OMR installs and the other for upgrading from existing postgres DB to SQLite. We rely on python's [sqlite3](https://docs.python.org/3/library/sqlite3.html) module to setup the SQLite backed database

### Scenario 1
#### SQLite database for fresh OMR installs

1. Configure a named volume mount on quay container to store sqlite data.
2. Modify the `DB_URI` parameters in the [config.yaml](https://github.com/quay/mirror-registry/blob/main/ansible-runner/context/app/project/roles/mirror_appliance/templates/config.yaml.j2#L7) to point to the new SQLite instance.
3. Remove references for postgres. This includes: 
    - Remove [ansible postgres](https://github.com/quay/mirror-registry/blob/main/ansible-runner/context/app/project/roles/mirror_appliance/tasks/install-postgres-service.yaml) task and it's [service](https://github.com/quay/mirror-registry/blob/main/ansible-runner/context/app/project/roles/mirror_appliance/templates/postgres.service.j2)
    - Remove postgres tarball creation from both Dockerfiles
    - Modify `mirror-registry` go binary to remove references to postgres (for both online and offline installs)

4. Write comprehensive tests to verify quay interaction with SQlite db and update CI tests.

### Scenario 2
#### DB Migration path from an existing postgres 10 to SQLite database.

For OMR running on postgres, the `./mirror-registry` binary will have a new command `migrate` that can be used to start the migration process. A new migration `sqlite-migration.yml` ansible task is defined which does the following: 
1. Brings down the `quay-app` service. 
2. Dump postgresSQL data: OMR stores db data in a named volume `pg-storage` in postgres container. Using `pg_dump` export this data into an `.sql` file.
3. On the quay repo, a python script is added to modify the `.sql` file. This includes:
  - Remove the lines starting with SET
  - Remove the lines starting with SELECT pg_catalog.setval
  - Replace true for ‘t’
  - Replace false for ‘f’
  - Add BEGIN; as first line and END; as last line
4. This script is configured to run when `quay-app` service is restarted in "sqlitemigration" mode handled via the [quay-entrypoint.sh](https://github.com/quay/quay/blob/master/quay-entrypoint.sh). During the restart, a SQlite volume is also mounted to the quay container which stores the modified `.sql` file.
5. Once the SQLite DB connection is successful, tasks are created in ansible to remove postgres instances. 


### Constraints

* Quay app will not be available during the entire migration process. 

### Risks and Mitigations

* 

### Test Plan

* TBD

### Implementation History

* 2023-11-28 Initial proposal
