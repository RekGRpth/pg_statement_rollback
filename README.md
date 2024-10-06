## Server side rollback at statement level for PostgreSQL

* [Description](#description)
* [Installation](#installation)
* [Configuration](#configuration)
* [Use of the extension](#use-of-the-extension)
* [Performances](#performances)
* [Problems](#problems)
* [Authors](#authors)
* [License](#license)

### [Description](#description)

pg_statement_rollback is a PostgreSQL extension to add server side
transaction with rollback at statement level like in Oracle or DB2.

If at any time during execution a SQL statement causes an error, all
effects of the statement are rolled back. The effect of the rollback
is as if that statement had never been run. This operation is called
statement-level rollback and has the following characteristics:

- A SQL statement that does not succeed causes the loss only of work
  it would have performed itself. The unsuccessful statement does
  not cause the loss of any work that preceded it in the current
  transaction.
- The effect of the rollback is as if the statement had never been
  run.

In PostgreSQL the transaction cannot continue when you encounter
an error and the entire work done in the transaction is rolled back.
Oracle or DB2 have implicit savepoint before each statement execution
which allow a rollback to the state just before the statement failure.

Current implementation of rollback at statement level for PostgreSQL
are done at client side. psql has `\set ON_ERROR_ROLLBACK on`, JDBC
has autorollback on SQL exception from executing a query, psqlODBC
too with the "statement level rollback" mode. The problem of these
implementations is that they add extra communication with the server
by sending a `SAVEPOINT autosave` and `RELEASE SAVEPOINT autosave`
so it can seriously limit the throughput of the application.

The pg_statement_rollback extension execute the automatic savepoint at
server side which adds a very limited penalty to the performances
(see "Performances" chapter below). Of course you still have to manage
the ROLLBACK TO SAVEPOINT at client side when an error occurs. For
example:

    BEGIN;
    CREATE TABLE savepoint_test(id integer);
    INSERT INTO savepoint_test SELECT 1;
    SELECT COUNT( * ) FROM savepoint_test; -- return 1
    INSERT INTO savepoint_test SELECT 'wrong 1'; -- generate an error
    -- Handle the error and fall back to previous statement.
    ROLLBACK TO "PgSLRAutoSvpt";
    SELECT COUNT( * ) FROM savepoint_test; -- still return 1
    ROLLBACK;

Without the extension everything will be cancelled and statement after
the error on INSERT will return:

    ERROR:  current transaction is aborted, commands ignored until end of transaction block

Here is the output of the test with statement-level rollback enabled:

    BEGIN
    CREATE TABLE
    INSERT 0 1
     count 
    -------
         1
    (1 row)
    
    psql:toto.sql:9: ERROR:  invalid input syntax for type integer: "wrong 1"
    LINE 1: INSERT INTO savepoint_test SELECT 'wrong 1';
                                              ^
    ROLLBACK
     count 
    -------
         1
    (1 row)
    
    ROLLBACK


### [Installation](#installation)

To install the pg_statement_rollback extension you need at least a
PostgreSQL version 9.5. Untar the pg_statement_rollback tarball
anywhere you want then you'll need to compile it with PGXS.  The
`pg_config` tool must be in your path.

Depending on your installation, you may need to install some devel
package. Once `pg_config` is in your path, do

    make
    sudo make install

To run test execute the following command as superuser:

    make installcheck

### [Configuration](#configuration)

#### Server side automatic savepoint

- *pg_statement_rollback.enabled*

The extension can be enable / disable using this GUC, default is
enabled. To disable the extension use:

    SET pg_statement_rollback.enabled TO off;

You can disable or enable the extension at any moment in a session.

- *pg_statement_rollback.savepoint_name*

By default the internal savepoint used is `PgSLRAutoSvpt` if you want
to change the name you can use this GUC. For example:

    SET pg_statement_rollback.savepoint_name TO 'my_new_sp_name';

then you will have to use `ROLLBACK TO SAVEPOINT my_new_sp_name;`.

This parameter can only be set by a superuser.

- *pg_statement_rollback.enable_writeonly*

By default the extension do not issue automatic savepoints after SELECT
statements. This is to limit the number of savepoints to avoid filling the
subtransaction cache. This behavior can be disabled without performances
penalty if you have less than 64 (PGPROC_MAX_CACHED_SUBXIDS) statements in
a transaction. Otherwise you will experience some performances lost due to
the on disk scan of pg_subtrans. 

Call to CTE or functions with nested write statements are detected and the
automatic savepoint is executed after the execution of the main statement. 

You can disable this feature by setting this directive to off. For example
if you are calling custom C functions that are writing directly into tables
that are not detected as write statements.


### [Use of the extension](#use-of-the-extension)

In all session where you want to use pg_statement_rollback transaction with
rollback at statement level you will have to load the extension using:

* LOAD 'pg_statement_rollback.so';
* SET pg_statement_rollback.enabled TO on;

Then in your application when an error is thrown you will have to call

    ROLLBACK TO SAVEPOINT "PgSLRAutoSvpt";

to continue the running transaction at the state just before the error.

If you want to generalize the use of the extension modify your postgresql.conf
to set

    session_preload_libraries = 'pg_statement_rollback'

and add

    pg_statement_rollback.enabled = on

at end of the file.

See files in test/sql/ for some examples of use.


### [Performances](performances)

When `log_min_duration_statement` is set to 0 the automatic savepoint
is always traced in the PostgreSQL log file with an arbitrary 0.01ms
which can correspond to a mean of their execution time. If necessary
the real timing could be added in the future.

To see the real overhead of loading the extension here is some pgbench
in tpcb-like scenario, best of three runs.

* Without loading the extension

```
$ pgbench -h localhost bench -c 20 -j 8 -T 30
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 8
duration: 30 s
number of transactions actually processed: 22298
latency average = 26.932 ms
tps = 742.603558 (including connections establishing)
tps = 742.811083 (excluding connections establishing)
```

* With the use of the extension and write only mode enabled

```
$ pgbench -h localhost bench -c 20 -j 8 -T 30
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 8
duration: 30 s
number of transactions actually processed: 21679
latency average = 27.702 ms
tps = 721.980032 (including connections establishing)
tps = 722.188840 (excluding connections establishing)
```

* With the use of the extension and write only mode disabled

```
$ pgbench -h localhost bench -c 20 -j 8 -T 30
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 8
duration: 30 s
number of transactions actually processed: 21677
latency average = 27.705 ms
tps = 721.903735 (including connections establishing)
tps = 722.126272 (excluding connections establishing)
```

Actually the pgbench scenario used here is not useful to test the interest
of limiting savepoint to write statements only. A special script should
be built with more than 64 (PGPROC_MAX_CACHED_SUBXIDS) statements in a
transaction to start playing with bottlenecks.

### [Problems](#problems)

When compiled with assert enabled (`--enable-cassert`) PostgreSQL will crash
when the extension is used. At line 1327 of ./src/backend/tcop/pquery.c the
following assert fail:

```
/*
 * Clear subsidiary contexts to recover temporary memory.
 */
Assert(portal->portalContext == CurrentMemoryContext);
```

Actually with the extension the memory context is not CurrentMemoryContext
as expected.

```
(gdb) b pquery.c:1327
Breakpoint 1 at 0x55792fd7a04d: file pquery.c, line 1327.
(gdb) c
Continuing.

Breakpoint 1, PortalRunMulti (portal=portal@entry=0x5579316e3e10, isTopLevel=isTopLevel@entry=true, 
    setHoldSnapshot=setHoldSnapshot@entry=false, dest=dest@entry=0x557931755ce8, altdest=altdest@entry=0x557931755ce8, 
    qc=qc@entry=0x7ffc4aa1f8a0) at pquery.c:1327
1327			Assert(portal->portalContext == CurrentMemoryContext);
(gdb) p portal->sourceText
$1 = 0x557931679c80 "INSERT INTO savepoint_test SELECT 1;"
(gdb) p MemoryContextStats(portal->portalContext)
$2 = void
(gdb) 
```
The memory context dump output
```
PortalContext: 1024 total in 1 blocks; 704 free (1 chunks); 320 used: <unnamed>
Grand total: 1024 bytes in 1 blocks; 704 free (1 chunks); 320 used
```

Clearly the assert in pquery.c doesn't allow our particular use for server side
statement-level rollback, PostgreSQL code should probably be modified because
we don't have possibilities to fix that at the extension level.

Here how to reproduce the crash:

```
SELECT pg_backend_pid();
LOAD 'pg_statement_rollback.so';
SET pg_statement_rollback.enabled = 1;
SET client_min_messages TO LOG;
SET log_statement TO 'all';
BEGIN;
CREATE TABLE savepoint_test(id integer);
-- run gdb on pid displayed above then => b pquery.c:1327
INSERT INTO savepoint_test SELECT 1; -- crash
```

### [Authors](#authors)

- Julien Rouhaud
- Dave Sharpe
- Gilles Darold


### [License](#license)

This extension is free software distributed under the PostgreSQL
License.

    Copyright (c) 2020-2024 LzLabs, GmbH

