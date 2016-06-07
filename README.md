Cassandra Plugins for Akka Persistence
======================================

[![Join the chat at https://gitter.im/akka/akka-persistence-cassandra](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/akka/akka-persistence-cassandra?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Replicated [Akka Persistence](http://doc.akka.io/docs/akka/2.4.3/scala/persistence.html) journal and snapshot store backed by [Apache Cassandra](http://cassandra.apache.org/).

[![Build Status](https://travis-ci.org/akka/akka-persistence-cassandra.svg?branch=master)](https://travis-ci.org/akka/akka-persistence-cassandra)

Dependencies
------------

### Latest release

To include the latest release of the Cassandra plugins into your `sbt` project, add the following lines to your `build.sbt` file:

    libraryDependencies += "com.typesafe.akka" %% "akka-persistence-cassandra" % "0.16"

This version of `akka-persistence-cassandra` depends on Akka 2.4.7 and Scala 2.11.8. 

It is compatible with Cassandra 3.0.0 or higher, and it is also compatible with Cassandra 2.1.6 or higher (versions < 2.1.6 have a static column bug) if you configure `cassandra-journal.cassandra-2x-compat=on` in your `application.conf`. With Cassandra 2.x compatibility some features will not be enabled, e.g. `eventsByTag`.

It implements the following [Persistence Queries](http://doc.akka.io/docs/akka/2.4.3/scala/persistence-query.html):

* allPersistenceIds, currentPersistenceIds
* eventsByPersistenceId, currentEventsByPersistenceId
* eventsByTag, currentEventsByTag (only for Cassandra 3.x)

Migrations
----------

### Migrations from 0.11 to 0.12

Dispatcher configuration was changed, see [reference.conf](https://github.com/akka/akka-persistence-cassandra/blob/v0.16/src/main/resources/reference.conf):

### Migrations from 0.9 to 0.10

The event data, snapshot data and meta data are stored in a separate columns instead of being wrapped in blob. Run the following statements in `cqlsh`:

    drop materialized view akka.eventsbytag;
    alter table akka.messages add writer_uuid text;
    alter table akka.messages add ser_id int;
    alter table akka.messages add ser_manifest text;
    alter table akka.messages add event_manifest text;
    alter table akka.messages add event blob;
    alter table akka_snapshot.snapshots add ser_id int;
    alter table akka_snapshot.snapshots add ser_manifest text;
    alter table akka_snapshot.snapshots add snapshot_data blob;

### Migrations from 0.6 to 0.7

Schema changes mean that you can't upgrade from version 0.6 for Cassandra 2.x of the plugin to the 0.7 version and use existing data without schema migration. You should be able to export the data and load it to the [new table definition](https://github.com/akka/akka-persistence-cassandra/blob/v0.16/src/main/scala/akka/persistence/cassandra/journal/CassandraStatements.scala#L25).

### Migrating from 0.3.x (Akka 2.3.x)

Schema and property changes mean that you can't currently upgrade from 0.3 to 0.4 SNAPSHOT and use existing data without schema migration. You should be able to export the data and load it to the [new table definition](https://github.com/akka/akka-persistence-cassandra/blob/v0.9/src/main/scala/akka/persistence/cassandra/journal/CassandraStatements.scala).

Journal plugin
--------------

### Features

- All operations required by the Akka Persistence [journal plugin API](http://doc.akka.io/docs/akka/2.4.3/scala/persistence.html#journal-plugin-api) are fully supported.
- The plugin uses Cassandra in a pure log-oriented way i.e. data are only ever inserted but never updated (deletions are made on user request only).
- Writes of messages are batched to optimize throughput for `persistAsync`. See [batch writes](http://doc.akka.io/docs/akka/2.4.3/scala/persistence.html#batch-writes) for details how to configure batch sizes. The plugin was tested to work properly under high load.
- Messages written by a single persistent actor are partitioned across the cluster to achieve scalability with data volume by adding nodes.

### Configuration

To activate the journal plugin, add the following line to your Akka `application.conf`:

    akka.persistence.journal.plugin = "cassandra-journal"

This will run the journal with its default settings. The default settings can be changed with the configuration properties defined in [reference.conf](https://github.com/akka/akka-persistence-cassandra/blob/v0.16/src/main/resources/reference.conf):

### Caveats

- Detailed tests under failure conditions are still missing.
- Range deletion performance (i.e. `deleteMessages` up to a specified sequence number) depends on the extend of previous deletions
    - linearly increases with the number of tombstones generated by previous permanent deletions and drops to a minimum after compaction

These issues are likely to be resolved in future versions of the plugin.

Snapshot store plugin
---------------------

### Features

- Implements the Akka Persistence [snapshot store plugin API](http://doc.akka.io/docs/akka/2.4.3/scala/persistence.html#snapshot-store-plugin-api).

### Configuration

To activate the snapshot-store plugin, add the following line to your Akka `application.conf`:

    akka.persistence.snapshot-store.plugin = "cassandra-snapshot-store"

This will run the snapshot store with its default settings. The default settings can be changed with the configuration properties defined in [reference.conf](https://github.com/akka/akka-persistence-cassandra/blob/v0.16/src/main/resources/reference.conf):


