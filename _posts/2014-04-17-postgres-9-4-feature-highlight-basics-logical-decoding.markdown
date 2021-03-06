---
author: Michael Paquier
lastmod: 2014-04-17
date: 2014-04-17 7:18:27+00:00
layout: post
type: post
slug: postgres-9-4-feature-highlight-basics-logical-decoding
title: 'Postgres 9.4 feature highlight: Basics about logical decoding'
categories:
- PostgreSQL-2
tags:
- postgres
- postgresql
- 9.4
- open source
- database
- logical
- decoding
- replication
- upgrade
- rolling
- multimaster
- core
- module
- plugin
- decoder
---
The second huge feature coming in PostgreSQL 9.4 with [jsonb]
(postgresql-2/postgres-9-4-feature-highlight-indexing-jsonb/) is called
[logical decoding]
(http://www.postgresql.org/docs/devel/static/logicaldecoding.html). In
short, it is a new plugin facility that can be used to decode changes
that happen on a database and stream them to external sources. It can
be used for many things like replication, auditing or even online upgrade
solutions.

Logical decoding has been introduced in the core of PostgreSQL
incrementally with a set of features that could roughly be listed as follows:

  * Logical replication slots, similar to [physical slots]
(/postgresql-2/postgres-9-4-feature-highlight-replication-slots/) except
that they are attached to a single database.
  * WAL level "logical" in wal\_level, level of WAL generated by server
to be able to decode changes to the database into a coherent format.
  * Creation of a SQL interface to view the changes of a replication slot.
  * Extension of the replication protocol to support logical replication
(with particularly the possibility to provide a database name in parameter
"replication" of a connection string)
  * Addition of [REPLICA IDENTITY]
(http://www.postgresql.org/docs/devel/static/sql-altertable.html), a table
parameter to modify how updated and deleted tuple data is written to WAL.

Then, two new utilities are present to help users to grab an understanding
of how things work:

  * test\_decoding, providing an example of output plugin for decoding.
  * pg\_recvlogical, an example of utility that can be used to receive
changes from a logical replication slot.

Logical decoding introduces a lot of new concepts and features, making it
impossible to write everything in a single post. Remember however that
it is possible to customize the decoding plugin, or in this post
test\_decoding, and the remote source receiving the changes, pg\_recvlogical
in the case of this post. So for now, using what Postgres core offers, let's
see how to simply set up logical replication. First, be sure that the
following parameters are set in postgresql.conf:

    wal_level = logical
    max_replication_slots = 1

max\_replication\_slots needs to be at least 1. test\_decoding needs to be
installed as well on your server. In order to work, logical replicaton
needs first a logical replication slot, which can be created using
pg\_create\_logical\_replication\_slot like that. Providing a plugin
name is mandatory:

    =# SELECT * FROM pg_create_logical_replication_slot('my_slot', 'test_decoding');
     slot_name | xlog_position
    -----------+---------------
     my_slot   | 0/16CB0C0
    (1 row)

xlog_position corresponds to the XLOG position where logical decoding
starts. After its creation, it is listed in the system view
pg\_replication\_slots, it will be marked as active once a remote source
using a replication connection starts the logical decoding with this slot:

    =# SELECT * FROM pg_replication_slots;
    -[ RECORD 1 ]+--------------
    slot_name    | my_slot
    plugin       | test_decoding
    slot_type    | logical
    datoid       | 12393
    database     | postgres
    active       | f
    xmin         | null
    catalog_xmin | 1001
    restart_lsn  | 0/16CB088

The next step is to enable the utility consuming the decoded changes,
pg\_recvlogical, like that for example:

    pg_recvlogical -d postgres --slot my_slot --start -f -

Note that you need to connect to the database where the replication slot
has been created, in my case "postgres". This command will also make all
the decoded changes to be printed in stdout.

pg\_recvlogical provides as well options to create and drop slots, this
is not really mandatory but it is rather handy when testing logical
decoding on multiple slots.

OK, now you will be able to see the logical changes received by
pg\_recvlogical. Logical decoding cannot replicate DDL changes,
so a simple DDL like that:

    =# CREATE TABLE ab (a int);
    CREATE TABLE

Results in that at the logical data receiver level (in the case of
test\_decoding of course):

    BEGIN 1002
    COMMIT 1002

Now here is how an insertion is decoded:

    # SQL query
    =# INSERT INTO ab VALUES (1);
    INSERT 0 1
    # Logical data receiver
    BEGIN 1003
    table public.ab: INSERT: a[integer]:1
    COMMIT 1003

Similarly to physical slots, as long as a receiver has not consumed the
changes of a slot, WAL files will be retained in pg_xlog, so be careful
that you pg_xlog partition or disk does not get completely filled up!
This post shows only the top of the iceberg of this feature, and there
are really a lot of things to tell about it. So stay tuned! For the time
being, feel free to have a look at this very promising infrastructure.
