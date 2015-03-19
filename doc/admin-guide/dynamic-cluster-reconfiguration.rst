Dynamic Cluster Reconfiguration
===============================

[[add-table]]
=== Adding a Table

A table can be added at any time, using either of two methods:

* Use the Admin Server's HTTP service: follow the "Add a table" hyperlink
  at the bottom of the top-level page.
+
* Use the `brick_admin` CLI interface at the Erlang shell.  See
link:hibari-contributor-guide.en.html#add-a-new-table[Hibari Contributor's Guide,
"Add a New Table" section].

[[remove-table]]
=== Removing a Table

NOTE: The current Hibari implementation does not support removing
a table.

In theory, most of the work of removing a table is already done:
chains that are abandoned after a migration are shut down
* Brick pinger processes are stopped.
* Chain monitor processes are stopped.
* Bricks are stopped.
* Brick data directories are removed.

All that remains is to update the Admin Server's schema to remove
references to the table.

[[chain-length-change]]
=== Changing Chain Length (Changing Replication Factor)

The Hibari Admin Server manages each chain as an independent data
replication entity.  Though Hibari clients view multiple chains that
are associated with a single table, each chain is actually independent
of the other chains.  It is possible to change the length of one chain
without changing any others.  For long term operation, such
differences do not make sense.  But during short periods of cluster
reconfiguration, such differences are possible.

A chain's length is determined by specifying a list of bricks that are
members of that chain.  The order of the list specifies the exact
chain order when the chain is in the `healthy` state.  By adding or
removing bricks from a chain definition, the length of the chain can
be changed.

A chain is defined by the Erlang 2-tuple of
`{ChainName, ListOfBricks}`, where each brick in `ListOfBricks` is a
2-tuple `{BrickName, NodeName}`.  For example, a chain of length two
called `footab_ch1` could be defined as:

  {footab_ch1, [{footab1_ch1_b1, 'gdss1@box-a'}, {footab1_ch1_b1, 'gdss1@box-b'}]}

The current definition of all chains for table `TableName` can be
retrieved from the Admin Server using the
`brick_admin:get_table_chain_list()` function, for example:

  %% Get a list of all tables currently defined.
  > brick_admin:get_tables().
  [tab1]

  %% Get list of chains in 'tab1' as they are currently in operation.
  > brick_admin:get_table_chain_list(tab1).
  {ok,[{tab1_ch1,[{tab1_ch1_b1,'gdss1@machine-1'},
                  {tab1_ch1_b2,'gdss1@machine-2'}]},
       {tab1_ch2,[{tab1_ch2_b1,'gdss1@machine-2'},
                  {tab1_ch2_b2,'gdss1@machine-1'}]}]}

This above chain list for table `tab1` corresponds to the chain and
brick layout below.

.Table `tab1`: Two chains of length two across two Erlang nodes on two physical machines
svgimage::images/tab1-2x2[align="center", scaledwidth="70%"]

NOTE: To change the definition of a chain, use the
`change_chain_length/2` or `change_chain_length/3` functions.  For
documentation, see
link:hibari-contributor-guide.en.html#changing-chain-length[Hibari Contributor's Guide,
"Changing Chain Length" section]

NOTE: When specifying a new chain definition, at least one brick from
the current chain must be included.

// JWN: Is it dangerous to allow an admin the opportunity to NOT SPECIFY the head
// of the chain in the new chain definition or to SPECIFY only a brick
// that is under repair?  I guess I see an opportunity for some
// "dynamic" (and not just static) pre-conditions that should/could be
// checked FIRST before starting to execute the changes.

[[chain-change-same-algorithm]]
==== Chain changes: same algorithm, different tasks.

The same brick repair technique is used to handle all three of the
following cases:

* adding a brick to a chain
* brick failure
* removing a brick from a chain

==== Adding a brick to a chain

When a brick `B` is added to a chain, that brick is treated as if it
was a member of the chain that had crashed long ago and has now been
restarted.  The same repair algorithm is used to synchronize data on
brick `B` that is used to repair bricks that were formerly in service
but since crashed and restarted.  See xref:chain-repair[] for a
description of the Hibari repair mechanism.

==== Brick failure

If a brick fails, the Admin Server must remove it from the chain by
reordering the chain.  The general order of operations are:

1. Set new roles for the chain's bricks, starting from the end of the
   chain and working backward.
2. Broadcast the new chain membership to all Hibari clients.

If a Hibari client attempts to send an operation to a brick during
step #2 and before the new chain info from step #2 arrives, that
client may send the operation to the wrong brick.  Hibari servers will
automatically forward the query to the correct brick.  Due to network
latencies and asynchronous message passing, it is possible that the
query be forwarded multiple times before it arrives at the correct
brick.

Specific details of how chain replication handles brick failure can be
found in van Renesse and Schneider's paper, see xref:chains[] for
citation details.

===== Failure of a head brick

If the head brick fails, then the first middle brick is promoted to the
head role.  If there is no middle brick (i.e. the length of the chain
was two), then the tail brick is promoted to a standalone role (chain
length is one).

===== Failure of a tail brick

If the tail brick fails, then the last middle brick is promoted to the
tail role.  If there is no middle brick (i.e. the length of the chain
was two), then the head brick is promoted to a standalone role (chain
length is one).

[[failure-middle-brick]]
===== Failure of a middle brick

The failure of a middle brick requires the most complex recovery
procedure.

* Assume that the chain is three bricks: `A` -> `B` -> `C`.
** If the chain is longer (more bricks upstream of `A` and/or more
 bricks downstream of `C`), the procedure remains the same.
* Brick `C` is configured to have its upstream brick be `A`.
* Brick `A` is configured to have its downstream brick be `C`.
* The head of the chain (brick `A` or the head brick upstream of `A`)
  requests a log flush of all unacknowledged writes downstream.  This
  step is required to re-send updates that were processed by `A` but
  have not been received by `C` because of middle brick `B`'s
  failure.
* Brick `A` waits until it receives a write acknowledgment from the
  tail of the chain.  Once received, all bricks in the chain have
  synchronously written all items to their write-ahead logs in the
  correct order.

==== Removing a brick from a chain

Removing a brick `B` permanently from a chain is a simple operation.
Brick `B` is
handled the same way that any other brick failure is handled: the
chain is simply reconfigured to exclude `B`.  See
xref:chain-reordering-middle-brick-fails[] for an example.

IMPORTANT: When a brick `B` is removed from a chain, all data from
brick `B` will be deleted when the operation is successful.  At this
time, the API does not have an option to allow `B`'s data to be
preserved.

// JWN: Wah ... a typo could be very dangerous.  Delayed deletion of
// the data and/or some other protective mechanism could be helpful.

[[chain-migration]]
=== Chain Migration: Rebalancing Data Across Chains

There are several cases where it is desirable to rebalance data across
chains and bricks in a Hibari cluster:

* Chains are added or removed from the cluster
* Brick hardware is changed, e.g. adding extra disk or RAM capacity
* A change in a table's consistent hashing algorithm configuration
  forces data (by definition) to another chain.

The same technique is used in all of these cases: chain migration.
This mirrors the same design philosophy that's used for handling chain
changes (see xref:chain-change-same-algorithm[]): use the same
algorithm to handle multiple use cases.

==== Example: Migrating from three chains to four

[[chain-migration-3to4]]
.Chain migration from 3 chains to 4 chains
svgimage::images/chain-migration-3to4[align="center", scaledwidth="80%"]

In the example above, both the 3-chain and 4-chain configurations used
equal weighting factors.  When all chains use the same weighting
factor (e.g. 100), then the consistent hashing map in the ``before''
and ``after'' cases look something like the figure below.

[[migration-3to4]]
.Migration from three chains to four chains
svgimage::images/migration-3to4[align="center", scaledwidth="70%"]

It doesn't matter that chain #4's total area within the unit interval
is divided into three regions.  What matters is that chain #4's total
area is equal to the regions of the other three chains.

==== Example: Migrating from three chains to four with unequal weighting

The diagram xref:migration-3to4[] demonstrates how a migration would
work when all chains have an equal weighting factor, e.g. 100.  If
instead, the new chain had a weighting factor of only 50, then the
distribution of keys to each chain would look like this:

.Migration from three chains to four with unequal chain weighting factors
[options="header"]
|=========
| Chain Name | Total % of keys before/after migration | Total unit interval size before/after migration
| Chain 1 | 33.3% -> 28.6% | 100/300 -> 100/350
| Chain 2 | 33.3% -> 28.6% | 100/300 -> 100/350
| Chain 3 | 33.3% -> 28.6% | 100/300 -> 100/350
| Chain 4 | 0% -> 14.3% (4.8% in each of 3 regions) | 0/300 -> 50/350 (spread across 3 regions)
| Total   | 100% -> 100% | 300/300 -> 350/350
|=========

For the original three chains, the total amount of unit interval
devoted to those chains is (100+100+100)/350 = 300/350.  The 4th
chain, because its weighting is only 50, would be assigned 50/350 of
the unit interval.  Then, an equal amount of unit interval is taken
from the original chains and reassigned to chain #4, so (50/350)/3 of
the unit interval must be taken from each original chain.

==== Hotspot migration

With the lowest level API, it is possible to assign "hot" keys to
specific chains, to try to balance a handful of keys that are very
frequently accessed from a large number of keys that are very
infrequently accessed.  The table below gives an example that builds
upon xref:migration-3to4[].  We assume that our "hot" key is mapped
onto the unit interval at position 0.5.

.Consistent hashing lookup table with three chains of equal weight and a fourth chain with an extremely small weight
[options="header"]
|=========
| Unit interval start | Unit interval end | Chain name
| 0.000000            | 0.333333...       | Chain 1
| 0.333333...         | 0.5               | Chain 2
| 0.5                 | 0.500000000000001 | Chain 4
| 0.500000000000001   | 0.666666...       | Chain 2
| 0.666666...         | 1.0               | Chain 3
|=========

The table above looks almost exactly like the "Before Migration" half
of xref:migration-3to4[].  However, there's a very tiny "hole" that is
punched in chain #2's space that maps key hashes in the range of 0.5
to 0.500000000000001 to chain #4.

[[adding-removing-client-nodes]]
=== Adding/Removing Client Nodes

It is not strictly necessary to formally configure a list of all
Hibari client nodes that may use a Hibari cluster.  However,
practically speaking, it is useful to do so.

To bootstrap itself to be able to use Hibari servers, a Hibari client
must be able to:

1. Communicate with other Erlang nodes in the cluster.
2. Receive "global hash" information from the cluster's Admin
   Server.

To solve both problems, the Admin Server maintains a list of Hibari
client nodes.  (Hibari server nodes do not need this mechanism.)  For
each client node, a monitor process on the Admin Server polls the node
to see if the `gdss` or `gdss_client` application is running.  If the
client node is running, then problem #1 (connecting to other nodes in
the cluster) is automatically solved by using `net_adm:ping/1`.
Problem #2 is solved by the client monitor calling
`brick_admin:spam_gh_to_all_nodes/0`.

The Admin Server's client monitor runs approximately once per second,
so there may be a delay of up to a couple of seconds before a
newly-started Hibari client node is connected to the rest of the
cluster and has all of the table info required to start work.

When a client node goes down, an OTP alarm is raised until the client
is up and running again.

Two methods can be used to view and change the client node monitor
list:

* Use the Admin Server's HTTP service: follow the "Add/Delete a client
  node monitor" hyperlink at the bottom of the top-level page.
* Use the Erlang CLI to use these functions:
** `brick_admin:add_client_monitor/1`
** `brick_admin:delete_client_monitor/1`
** `brick_admin:get_client_monitor_list/0`
