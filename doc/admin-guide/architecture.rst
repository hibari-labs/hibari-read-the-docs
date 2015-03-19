Hibari Architecture
===================

From a logical point of view, Hibari's architecture has three layers:

- Top layer: **consistent hashing**
- Middle layer: **chain replication**
- Bottom layer: **the storage brick**

This section discusses each of these major layers in detail, starting
from the bottom and working upward.

.Logical architecture diagram; physical hosts/bricks are color-coded with 5 colors
svgimage::images/logical-architecture1[align="center", scaledwidth="80%"]

.Logical architecture diagram, alternative perspective
svgimage::images/logical-architecture-alt[align="center", scaledwidth="80%"]

Bricks, Physical and Logical
----------------------------

The word "brick" has two different meanings in a Hibari system:

- An entire physical machine that has Hibari software installed,
  configured, and (hopefully) running.
- A logical software entity that runs inside the Hibari application
  that is responsible for managing key-value pairs.

[[the-physical-brick]]

The physical brick
^^^^^^^^^^^^^^^^^^

The phrase "physical brick" and "machine" are interchangeable, most of
the time.  Hibari is designed to react correctly to the failure of any
part of the machine that the Hibari application is running:

- disk
- power supply
- CPU
- network

Hibari is designed to take advantage of low-cost, off-the-self
commodity servers.

A physical brick is the basic unit of failure. Data replication (via
the chain replication algorithm) is responsible for protecting data,
not redundant equipment such as dual power supplies and RAID disk
subsystems.  If a physical brick crashes for any reason, copies of
data on other physical bricks can still be used.

It is certainly possible to decrease the chances of data loss by using
physical bricks with more expensive equipment. Given the same number
of copies of a key-value pair, the chances of data loss are less if
each brick has multiple power supplies and RAID 1/5/6/10 disk. But
risk of data loss can also be reduced by increasing the number of data
replicas ("chain length") using cheaper, non-redundant server
hardware.

The logical brick
^^^^^^^^^^^^^^^^^

A logical brick is a software entity that runs within a Hibari
application instance on a physical brick. A single Hibari physical
brick can support dozens or (potentially) hundreds of logical bricks,
though limitations of CPU, RAM, and/or disk capacity can impose a
smaller limit.

A logical brick maintains RAM and disk data structures to store a
collection of key-value pairs. The keys are maintained in
lexicographic sorting order.

The replication technique used by Hibari, chain replication, maintains
identical copies of key-value pairs across multiple logical bricks.
The number of copies of a key-value pair is exactly equal to the
length of the chain. See the next subsection below for more details.

It is possible to configure Hibari to place all of the logical bricks
for the same chain onto the same physical brick. This practice can be
useful in a developer's environment, but it is impractical for
production networks: such a configuration does not have any physical
redundancy, and therefore it poses a greater risk of data loss.

[[write-ahead-logs]]

Write-Ahead Logs
^^^^^^^^^^^^^^^^

By default, all logical bricks will record all updates to a
**write-ahead log**. Used by many database systems, a write-ahead log
(WAL) appears to be an infinitely-sized log where all important events
(e.g. all write and delete operations) are appended to the end of the
log. The log is considered **write-ahead** if a log entry is written
*prior to* any significant processing by the application.

[[write-ahead-logs-in-hibari]]

Write-ahead logs in the Hibari application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Two types of write-ahead logs are used by the Hibari application.
These logs cooperate with each other to provide several benefits to
the logical brick.

There are two types of write-ahead logs:

- The shared **common log**. This single write-ahead log instance provides
  durability guarantees to all logical bricks within the server node
  via the ``fsync()`` system call.
- Individual **private logs**. Each logical brick maintains its own private
  write-ahead log instance. All metadata regarding keys in the
  logical brick are stored in the logical brick's private log.

All updates are written first to the common log, usually in a
synchronous manner. At a later time, update metadata is lazily copied
from the common log to the corresponding brick's private log.
Value blobs (for bricks that store value blobs on disk)
will remain in the common log and are managed by
the **scavenger**, see xref:scavenger[].

svgimage::images/private-and-common-logs[align="center", scaledwidth="80%"]

[[two-wal-types]]

Two types of write-ahead logs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The two log types cooperate to support a number of useful properties.

- Data durability in case of system crash or power failure. All
  synchronous writes to the ``common log'' are guaranteed to be
  flushed to stable storage.
- Performance enhancement by limiting ``fsync()`` usage. After a
  logical brick writes data to the common log, it will request an
  ``fsync()``. The common log will combine ``fsync()`` requests from
  multiple bricks into a single system call.
- Performance enhancement at logical brick startup. A brick's private
  log stores only that bricks key metadata. Therefore, at startup
  time, the logical brick does not scan data maintained by other
  logical bricks. This can be a very substantial time savings as the
  amount of metadata managed by all logical bricks grows over time.
- Performance enhancement by separating synchronous writes from
  asynchronous writes. If the common log's storage is on a separate
  device, e.g. a write-optimized flash memory block device, then all
  of the ``fsync()`` calls can finish much faster. During later
  processing of the asynchronous/lazy copying of key metadata from the
  common log to individual private logs can take advantage of OS dirty
  page write coalescing and other I/O optimizations without
  interference by ``fsync()``.  These copies are performed roughly once
  per second.

[[wal-dirs-and-files]]

Directories and files used by write-ahead logs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each write-ahead log is stored on disk as a collection of large files
(default = 100MB each).  Each file in the log is identified by a **log
sequence number** and is called a **log sequence file**.

Log sequence files are append-only and are never written again.
Consequently, data in a log sequence file is never overwritten. Any
disk space reclaimed by checkpoint and scavenger operations is done by
copying data from old log sequence files and appending to new log
sequence files. Once the new log sequence file(s) is flushed to
stable storage, the old log sequence file(s) can be deleted.

When a log sequence file reaches its maximum size, the current log
file is closed and a new one is opened with a monotonically increasing
log serial number.

All log files for a write-ahead log are grouped under a single
directory called, ``hlog.{log-name}``, where ``{log-name}`` is the
name of the brick or of the common log.  These directories are stored
under the ``var/data`` subdirectory of the application's installation
path, ``/usr/local/TODO/TODO/var/data`` (by default).

The maximum log file size (``brick_max_log_size_mb`` in the
``central.conf`` file) is advisory only and is not enforced as a hard
limit.

Reclaiming disk space used by write-ahead logs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In practice, infinite storage is not yet available. The Hibari system
uses two mechanisms to reclaim unused disk space:

- The **checkpoint** mechanism, see xref:checkpoints[].
- The **scavenger** mechanism, see xref:scavenger[].

Write-ahead log serial numbers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each item written in a write-ahead log is assigned a serial number.
If the brick is in **standalone** or **head** roles, then the serial
number will be assigned by that brick. For downstream bricks, the
serial number assigned by the **head** brick will be used.

The serial number mechanism is used to ensure that a single unique
ordering of log items will be written to each brick log.  In certain
failure cases, log items may be re-sent down the chain a second time,
see xref:failure-middle-brick[].

// JWN: Does the above mechanism "to ensure that a single unique ordering"
// applies to both common log and private log?

[[chains]]
=== Chains

A chain is the unit of data replication used by the
link:http://www.usenix.org/events/osdi04/tech/renesse.html[``chain replication'' technique as described in this paper]:

  Chain Replication for Supporting High Throughput and Availability
  Robbert van Renesse and Fred B. Schneider
  USENIX OSDI 2004 conference proceedings
  http://www.usenix.org/events/osdi04/tech/renesse.html

Data replication algorithms can be separated into two basic families:

* State machine replication
* Quorum replication

The chain replication algorithm is from the state machine family of
replication algorithms.  It is a variation of the familiar
``master/slave'' replication algorithm, where all updates are sent to
a master node and then copies are sent to zero or more slave nodes.

Chain replication requires a very specific ordering of nodes (which
store copies of data) and the messages passed between them.  The
diagram below depicts the "key update" message flow in a chain of
length three.

[[diagram-write-path-3]]
.Message flow in a chain for a key update
svgimage::images/write-path-3[align="center", scaledwidth="80%"]

If a chain is of length one, then the same brick assumes both ``head''
and ``tail'' roles simultaneously.  In this case, the brick is called
a ``standalone'' brick.

.Message flow for a key update to a chain of length 1
svgimage::images/write-path-1[align="center", scaledwidth="30%"]

To maintain the property strong consistency, a client must read data
from the tail brick in the chain.  A read processed by any other brick
member would permit the client to read an update that has not yet been
processed by all bricks and therefore could result in a strong
consistency violation.  Such a violation is frequently called a
``dirty read'' in other database systems.

.Message flow for a read-only key query
svgimage::images/read-path-3[align="center", scaledwidth="80%"]

[[bricks-outside-chain-replication]]
==== Bricks outside of chain replication

During Hibari's development, we encountered a problem of managing the
state required by the Admin Server.  If data managed by chain
replication requires the Admin Server to be running, how can the Admin
Server read its own data?  There is a ``chicken and the egg''
dependency problem that must be solved.

// JWN: Why wasn't Mnesia used for the Admin Server's storage
// implementation?

The solution is simple: do not use chain replication to manage the
Admin Server's data.  Instead, that data is replicated using a simple
``quorum replication'' technique.  These bricks all have names starting
with the string "bootstrap".

A brick must be in ``standalone'' mode to answer queries when it is
used outside of chain replication.  See xref:brick-roles[] for details
on the standalone role.

=== Tables

A table is thing that divides the key namespace within Hibari.  If you
need to have two different keys called "foo" but have different
values, you store each "foo" key in a separate table.  The same is
true in other database systems.

Hibari's implementation uses one or more replication chains to store
the data for one table.

.Relationship between tables, chains, and bricks.
svgimage::images/table-chain-brick[align="center", scaledwidth="70%"]

[[micro-transactions]]
=== Micro-Transactions

In a single request, a Hibari client may send multiple update
operations to the cluster.  The client has the option of requesting
``micro-transaction'' semantics for those updates: if there are no
errors, then all updates will be applied atomically.  This behaves
like the ``transaction commit'' behavior supported by most relational
databases.

On the other hand, if there is an error while processing one of the
update operations, then all of update operations will fail.  This
behaves like the ``transaction abort'' behavior supported by most
relational databases.

Unlike most relational databases, Hibari does not have a transaction
manager that can coordinate ACID semantics for arbitrary read and
write operations across any row in any table.  In fact, Hibari has no
transaction manager at all.  For this reason, Hibari calls its limited
transaction feature ``micro-transactions'', to distinguish this
feature from other database systems.

Hibari's micro-transaction support has two important limitations:

* All keys involved in the transaction must be stored in the same
  replication chain (and therefore by the same brick(s)).
* Operations within the micro-transaction cannot see updates by other
  operations within the the same micro-transaction.

.Four keys in the "footab" table, distributed across two chains of length three.
[id="footab-example"]
svgimage::images/micro-transaction-example[align="center", scaledwidth="70%"]

In the diagram above, a micro-transaction can be permitted if it
operates on only the keys "string1" & "string4" or only the keys
"string2" and "string3".  If a client were to send a micro-transaction
that operates on keys "string1" and "string3", the micro-transaction
will be rejected: key "string3" is not stored by the same chain as the
key "string1".

.Valid micro-transaction: all keys managed by same chain
[id="valid-utxn"]
    [txn,
     {op = replace, key = "string1", value = "Hello, world!"},
     {op = delete, key = "string4"}
    ]

.Invalid micro-transaction: keys managed by different chains
[id="invalid-utxn"]
    [txn,
     {op = replace, key = "string1", value = "Hello, world!"},
     {op = delete, key = "string2"}
    ]

The client does not have direct control over how keys are distributed
across chains.  When a table is defined and created, its configuration
specifies the algorithm used to map a {TableName, Key} pair to a
specific chain.

// JWN: This might be a good place to briefly explain the benefits of
// using a key prefix and how it is beneficial to (some) applications.

NOTE: See
link:hibari-contributor-guide.en.html#add-a-new-table[Hibari Contributor's Guide,
"Add a New Table" section]
for more information about table configuration.

=== Distribution: Workload Partitioning and Fault Tolerance

[[consistent-hashing-example]]
==== Partitioning by consistent hashing

To spread computation and storage workloads across all servers in the
cluster, Hibari uses a technique called ``consistent hashing''.  This
hashing technique attempts to distribute a table's key space evenly
across all chains used by that table.

IMPORTANT: The word ``consistent'' has slightly different meanings
relative to ``consistent hashing'' and ``strong consistency''.  The
consistent hashing algorithm is a commonly-used algorithm for key ->
storage location calculations.  Consistent hashing does not affect the
``eventual consistency'' or ``strong consistency'' semantics of a
database system.

See the xref:footab-example[] for an example of a table with two
chains.

See
link:hibari-contributor-guide.en.html#add-a-new-table[Hibari Contributor's Guide,
"Add a New Table" section]
for details on valid options when creating new tables.

===== Consistent hashing algorithm

Hibari uses the following steps in its consistent hashing algorithm
implementation:

* Calculate the ``hashing prefix'', using part or all of the key as
  input to the next step.
** This step is configurable, using built-in functions or by providing
   a custom implementation function.
** Built-in prefix functions:
*** Null: use entire key
*** Fixed length, e.g. 4 byte or 8 byte constant length prefix.
*** Variable length: use separator character `'/'` (configurable)
    such that hash prefix is found between the first two (also
    configurable) `'/'` characters.  E.g. If the key is `/user/bar`,
    then the string `/user/` is used as the hash prefix.
* Calculate the MD5 checksum of the hashing prefix and then convert
  the result to the unit interval, 0.0 - 1.0, using floating point
  arithmetic.
* Consult the unit interval -> chain map to calculate the chain name.
** This map contains a tree of `{StartValue, EndValue, ChainName}` tuples.
   For example, `{0.0, 0.5, footab_ch1}` will map the interval
   `(0.0, 0.5]` to the chain named `footab_ch1`.
*** The mapping tree's construction is affected by the chain weighting
    factor.  The weighting factor allows some chains to store more
    than other chains.
* Use the operation type to calculate the brick name.
** For read-only operations, choose the tail brick.
** For update operations, choose the head brick.

===== Consistent hashing algorithm use within the cluster

* Hibari clients use the algorithm to calculate which chain must
handle operations for a key.  Clients obtain this information via
updates from the Hibari Admin Server.  These updates allow the client
to send its request directly to the correct server in most use cases.

* Servers use the algorithm to verify that the client's calculation
was correct.
** If a client sends an operation to the wrong brick, the brick will
forward the operation to the correct brick.
** If a client sends a list of operations such that some bricks are
stored on the brick and other keys are not, an error is returned to
the client.  Micro-transactions are not supported across chains.

===== Changing consistent hashing configuration dynamically

Hibari's Admin Server will allow changes to the consistent hashing
algorithm without service interruption.  Such changes are applied on a
per-table basis:

* Adding or removing chains to the unit interval -> chain map.
* Modifications of the chain weighting factor.
* Modifying the key -> hashing prefix calculation function.

See the xref:chain-migration[] section for more information.

==== Multiple replicas for fault tolerance

For fault tolerance, data replication is required.  As explained in
xref:chains[], the basic unit of failure is the brick.  The chain
replication algorithm will maintain replicas of keys in a strongly
consistent manner across all bricks: head, middle, and tail bricks.

To be able to tolerate `F` failures without data loss or service
interruption, each replication chain must be at least `F+1` bricks
long.  This is in contrast to quorum replication family algorithms,
which typically require `2F+1` replica bricks.

// JWN: Would it be helpful to put a note that typically "3" is the
// recommended number of replicas?

===== Changing chain length configuration dynamically

Hibari's Admin Server will allow changes to a chain's length without
service interruption.  Such changes are applied on a per-chain basis.
See the xref:chain-length-change[] section for more information.
