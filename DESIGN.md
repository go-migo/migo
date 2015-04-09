
### Goal
migo is a tool for migrating SQL database schema's in a running environment. It allows developers to modify a database schema with minimum downtime or service interruption. Services that connect to the database become aware of schema upgrades and can automatically update their statements when an update occurred.

### Overview
migo combines traditional upgrade sql files with etcd to create a system where upgrades can be rolled out with automated database migration.

    usage layout
    .
       /---> sql db <---\
       |                |
    amigo <-> etcd <-> your application (migo package)
                ^
                |
             migoctl <-> local repository

 - etcd is used as key-value store for data storage and locking/registration.
 - the local repository holds upgrade.sql files
 - migoctl pushes local files to etcd
 - amigo executes upgrade scripts on the sql database
 - applications using the migo package automatically update their sql queries after the schema was upgraded

#### "features"
Features refer to a subset of the database schema. Each feature has a version number which is a full integer (like the upgrade revision). A revision can increment feature versions or add a new feature (version 1). New feature versions can be 'breaking', which adds extra checks and a temporary hold on all programs using the feature while performing the upgrade.

### Local migo repository
A migo repository contains the files for a migo-enforced database schema.

Repository structure:

 - `migo.yaml` -- migo project configuration (environment-agnostic)
 - `revisions/` -- directory containing the revision upgrades
 - `revisions/000/` -- create database statement
 - `revisions/001/` -- first upgrade (first create table statements)
 - `revisions/002/` -- second upgrade
 - `revisions/etc...`

An upgrade revision contains the following files:

 - `revision.yaml` -- metadata about the upgrade
 - `upgrade.sql` -- schema upgrade statements
 - `devdata.sql` -- (optional) development data to be executed after the upgrade (only in development and/or testing environment)

#### migo.yaml
The migo yaml file contains some parameters such as project name and description. It does not contain environment specific values, those values are "alive" in each environment. It can contain defaults used for all environments.

#### revisions/
This directory contains only directories who's names are valid decimal positive integers (can have leading zero's).

#### revision.yaml
Revision information parameters.

 - features: array of objects having fields `name string`, `version int`, `breaking bool`

#### upgrade.sql
Schema modifications. Can also include data migration statements.

#### devdata.sql
The `devdata.sql` file can contain `insert`, `update` and `delete` statements. These are executed in environments with the `devdata` parameter set.

### amigo
`amigo` is the migo daemon. It obtains etcd locks, executes upgrade statements, and updates the state in etcd. Multiple instances can be ran for failover pusposes, but only one instance will be active.

#### Upgrade execution
When starting an upgrade, amigo makes sure that no migo-enabled applications hold a lock on any of the features that are going to be updated in a breaking way. If there is one or more locks, amigo schedules an upgrade time ahead of those locks and waits for all the locks to clear. It then executes the upgrade en clears the upgrade time afterwards, allowing migo-enabled applications to obtain locks on the (upgraded) feature again.

The SQL statements in the upgrade.sql files are split into separate statements by semicolon. Statements are executed separately in a database transaction.

### migoctl
`migoctl` is a tool to administrate migo setups, it provides the following commands:

 - `migoctl repo init` -- create a local migo repository with mock files
 - `migoctl repo list-features <n>|latest` -- list the features with their versions
 - `migoctl repo new-revision` -- add a new revision folder
 - `migoctl push` -- push repo to a remote etcd
 - `migoctl remote upgrade <n>|latest|one` -- request an upgrade up to revision `n`, the latest revision, or one revision.
 - `migoctl remote reset` -- reset fault/error status and set `revision.requested` back to `revision.current`.
 - `migoctl remote drop` -- drop the database structure
 - `migoctl remote status` -- get status from etcd (amigo state)
 - `migoctl remote config <key> [<value>]` -- set config parameters in etcd

When communicating with etcd, `migoctl` requires the etcd endpoint to be passed by the `--endpoint`. Alternatively, `--endpoint` can be provided through the `MIGO_ENDPOINT` environment variable. `migoctl` does not assume a default endpoint. Example: `migoctl push --endpoint http://12.34.56.78:4001`.

### Compatibility
TODO: notes on when a database schema is non-breaking.

### Notes to self
 - migo uniqueid in etcd and migo.yaml, must match, otherwise cancel push
 - allow post-upgrade hooks by applications using `pkg migo`.
