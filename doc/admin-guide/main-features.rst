Hibari's Main Features in Broad Detail
======================================

=== Distributed system

Multiple machines can participate in a single cluster.  The maximum
size of a Hibari cluster has not yet been determined.  A practical
limit of approximately 200-250 nodes is likely.

Any server node can handle any client request, forwarding a request to
the correct server node when necessary.  Clients maintain enough state
to send their queries directly to the correct server node in all
common cases.

=== Scalable system

The total storage and processing capacity of a Hibari cluster
increases linearly as machines are added to the cluster.

=== Durable updates

Every key update is written and flushed to stable storage (via the
`fsync()` system call) before sending acknowledgments to the client.

=== Consistent updates

After a key's update is acknowledged, no client in the cluster can see
an older version of that key.  Hibari uses the "chain replication"
algorithm to maintain consistency across all replicas of a key.

All data written to disk include MD5 checksums; the checksums are
validated on each read to avoid sending corrupted data to the client.

[[lockless-client-api]]
=== Lockless client API

The Hibari client API requires that all operations (read queries
operations and/or update operations) be self-contained within a single
client request.  Therefore, locks are not implemented because they are
not required.

Inside Hibari, each key-value pair also contains a ``timestamp''
value.  A timestamp is an integer.  Each time the key is updated, the
timestamp value must increase.  (This requirement is enforced by all
server nodes.)

In many database systems, if a client requires guarantees that a key has
not changed since the last time it was read, then the client acquires
a lock (or lease) on the key.  In Hibari, the client's update
specifies the timestamp of the last read attempt of the key:

* If the timestamp matches the server, the operation is permitted.
* If the timestamp does not match the server's timestamp, then the
  operation is not permitted, and the new timestamp is returned to the
  client.

It is recommended that all Hibari nodes use NTP to synchronize their
system clocks.  The simplest Hibari client API uses timestamps based
upon the OS system clock for timestamp values.  This feature can be
bypassed, however, by using a slightly more complex client API.

However, Hibari's overload detection and work-dumping algorithms will
use the OS system clock, regardless of which client API is used.  All
system clocks, client and server, be synchronized to be within roughly
1 second of each other.

=== High availability

Each key can be replicated multiple times (configurable on a per-table
basis).  As long as one copy of the key survives, all operations on
that key are permitted.  A cluster can survive multiple cluster node
failures and still maintain full data integrity.

The cluster membership application, called the Hibari Admin Server,
runs as an active/standby application on one or more of the server
nodes.  The Admin Server's configuration and private state are also
maintained in Hibari server nodes.  Shared storage such as NFS, shared
SCSI/Fibre Channel LUNs, or replicated block devices are not required.

If the Admin Server fails and is restarted on a standby node, the rest
of the cluster can continue normal operation.  If another brick fails
while the Admin Server is restarting, then clients may see service
interruptions (usually in the form of timeouts) until the Admin Server
has finished restarting and can react to the failure.

=== Multiple Client Protocols

Hibari supports many client protocols for queries and updates:

* A native Erlang API, via Erlang's native message-passing mechanism
* Amazon S3 protocol, via HTTP
* UBF, Joe Armstrong's ``Universal Binary Format'' protocol, via TCP
* UBF via several minor variations of TCP transport
* UBF over JSON-RPC, via HTTP
* JSON-encoded UBF, via TCP

Protocols under development:

* Memcached, via TCP
* UBF over Thrift, via TCP
* UBF over Protocol Buffers, via TCP

Most of the client access protocols are implemented using the
Erlang/OTP `application` behavior.  By separating each access protocol
into separate OTP applications, Hibari's packaging is quite flexible:
packaging can add or remove protocol support as desired.  Similarly,
protocols can be stopped and started at runtime.

[[overview-high-performance]]
=== High performance

Hibari's performance is competitive with other distributed,
non-relational databases such as HBase and Cassandra, when used with
similar replication and durability configurations.  Despite the
constraints of durable writes and strong consistency, Hibari's
performance can exceed those databases on some workloads.

IMPORTANT: The metadata of all keys stored by the brick, called the
``key catalog'', are stored in RAM to accelerate commonly-used
operations.  In addition, non-zero values of the "expiration_time" and
non-empty values of "flags" are also stored in RAM (see
xref:sql-definition-hibari[]).  As a consequence, a multi-million key
brick can require many gigabytes of RAM.

=== Automatic repair

Replicas of keys are automatically repaired whenever a cluster node
crashes and restarts.

=== Dynamic configuration

The number of replicas per key can be changed without service
interruption.  Likewise, replication chains can be added or removed
from the cluster without service interruption.  This permits the
cluster to grow (or shrink) as workloads change.  See
xref:chain-migration[] for more details.

=== Data rebalancing

Keys will be automatically be rebalanced across the cluster without
service interruption.  See xref:chain-migration[] for more details.

=== Heterogeneous hardware support

Each replication chain can be assigned a weighting factor that will
increase or decrease the percentage of a table's key space relative to
all other chains.  This feature can permit use of cluster nodes with
different CPU, RAM, and/or disk capacities.

=== Micro-Transactions

Under limited circumstances, operations on multiple keys can be given
transactional commit/abort semantics.  Such micro-transactions can
considerably simplify the creation of robust applications that keep
data consistent despite failures by both clients and servers.

[[per-table-config-perf-options]]
=== Per-table configurable performance options

Each Hibari table may be configured with the following options to
enhance performance ... though each of these options has a
corresponding price to pay.

* RAM-based storage: All data (both keys and values) may be stored in
  RAM, at the expense of increased RAM consumption.
  Disk is used still used to log all updates, to protect against
  a catastrophic power failure.
* Asynchronous writes: Use of the `fsync()` system call can be disabled
  to improve performance, at the expense of data loss in a system
  crash or power failure.
* Non-durable updates: All update logging can be disabled to improve
  performance, at the expense of data loss when all nodes in a
  replication chain crash.
