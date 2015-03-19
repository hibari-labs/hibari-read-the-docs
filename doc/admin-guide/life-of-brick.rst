The Life of a (Logical) Brick
=============================

All logical bricks within a Hibari cluster go through the same set of
lifecycle events.  Each is described in greater detail in this
section.

* Brick initialization and operation states, described by a finite
  state machine.
* Brick roles within chain replication, also described by a finite
  state machine.
* Periodic housekeeping tasks performed by logical bricks and their
  internal support services, e.g. checkpoints and the ``scavenger''.

[[brick-lifecycle-fsm]]
=== Brick Lifecycle Finite State Machine

The lifecycle of each Hibari logical brick goes through a set of
states defined by a finite state machine (OTP `gen_fsm` behavior) that
is executed by a process within the Admin Server application.

.Logical brick lifecycle finite state machine
svgimage::images/brick-fsm[align="center"]

.Logical brick lifecycle FSM states
unknown;;
  This is the initial state of the FSM.  Because the Admin Server may
  crash or be restarted at any time, this state is used by the Admin
  Server when it has not been running long enough to determine the
  state of the logical brick.
pre_init;;
  A brick moves itself to this state when it has finished scanning its
  private write-ahead log (see xref:write-ahead-logs[]) and therefore
  knows the state of all keys that it manages.
repairing;;
  In chain replication, the repairing state is used to synchronize a
  a newly started/restart brick with the rest of the chain.  At the
  end of this state, the brick is 100% in sync with all other active
  members of the chain.  Repair is initiated by the Admin Server's
  chain monitor that is responsible for the chain.
ok;;
  The brick moves itself to this state when repair has finished.  The
  brick is now in service and capable of servicing Hibari client
  requests.  Client requests will be rejected if the brick is in any
  other state.
  * If managed by chain replication, this brick is eligible to be put
    into service as a full member of a replication chain.
    See xref:brick-roles[].
  * If managed by quorum replication, some external entity must change
    the logical brick's state from `pre_init` -> `ok`.  Hibari's Admin
    Server automates this task for the `bootstrap_copy`* bricks.
    The present implementation of the Admin Server does not manage
    quorum replication bricks outside of the Admin Server's private
    use.
disk_error;;
  A disk error has occurred, for example a missing file or directory
  or MD5 checksum error.  Administrator intervention is required to
  move a brick out of the `disk_error` state: shut down the entire
  Hibari server, kill the logical brick manually, or use the
  brick_chainmon:force_best_first_brick() function manually.

[[chain-lifecycle-fsm]]
=== Chain Lifecycle Finite State Machine

The chain FSM (OTP `gen_fsm` behavior) is executed by a process within
the Admin Server application.  All state transitions are triggered by
changes in the state of each member bricks' state, into or out of the
`'ok'` state.  See xref:brick-lifecycle-fsm[] for details.

.Chain replication finite state machine
svgimage::images/chain-fsm[align="center"]

.Chain lifecycle FSM states
unknown;;
  The state of the chain is unknown. Information regarding chain members is
  unavailable. Because the Admin Server may
  crash or be restarted at any time, this state is used by the Admin
  Server when it has not been running long enough to determine the
  state of the chain.
  It is possible that the chain was in `degraded` or `healthy` state
  before the crash and therefore Hibari client operations may be
  serviced while in this state.
unknown_timeout;;
  This intermediate state is used by the Admin Server before moving
  automatically to another state.
stopped;;
  All bricks in the chain are crashed or believed to have crashed.
  Service to Hibari clients will be interrupted.
degraded;;
  Some (but not all) bricks in the chain are in service.  The Admin
  Server will wait for another chain member to enter its `pre_init`
  state before chain repair can start.
healthy;;
  All bricks in the chain are in service.

[[brick-roles]]
=== Brick ``Roles'' Within A Chain

Each brick within a chain has a role.  The role will be changed by the
Admin Server whenever it detects that the chain's state has changed.
These roles are:

head;;
  The brick is first in the chain, i.e. at the ``head'' of the chain's
  ordered list of bricks.
tail;;
  The brick is last in the chain, i.e. at the ``tail'' of the chain's
  ordered list of bricks.
middle;;
  The brick is neither the ``head'' nor ``tail'' of the chain.
  Instead, the brick is somewhere in the middle of the chain.
standalone;;
  In a chain of length 1, the ``standalone'' brick is a brick that
  acts both as a ``head'' and ``tail'' brick simultaneously.

There is one additional attribute that is given to a brick in a
cluster.  Its name ``official tail''.

official tail;;
  The official tail brick has two duties for the chain:
  * It handles read-only queries to the chain.
  * It sends replies to the client for all update operations that are
  sent to the head of the chain.

////////
Hrm, using ``official tail'' in this paragraph doesn't work correctly, grrrr.
It flips to mono font after the closing ''.
////////
As far as Hibari clients are concerned, the chain member with the
the "official tail" role is the brick that they consider the "tail" of
the chain.  Hibari clients are not aware of "tail" bricks that are
undergoing repair.  Any client request that is sent to a `repairing`
state brick will be rejected.

See xref:diagram-write-path-3[] for an example of a healthy chain of
length three.

[[brick-init]]
=== Brick Initialization

A logical brick does not maintain an on-disk data structure, such as a
binary tree or B-tree, to keep track of the keys it stores.  Instead,
each logical brick maintains that metadata entirely in RAM.
Therefore, the only time that the metadata in the private write-ahead
log is ever read is at brick initialization time, i.e. when the brick
restarts.

The contents of the private write-ahead log are used to repopulate the
brick's ``key catalog'', the list of all keys (and associated
metadata) stored by the brick.

When a logical brick is started, all of the log sequence files in
the private log are read, starting from the oldest and ending with the
newest.  (See xref:wal-dirs-and-files[].)  The total amount of data
required at startup can be quite small or it can be hundreds of
gigabytes.  The factors that influence the amount of data in the
private log are:

* The total number of keys stored by the logical brick.
** More keys means that the log sequence file created by a checkpoint
   operation will be larger.
* The size of the `brick_check_checkpoint_max_mb` configuration parameter
  in the `central.conf` config file.

When the log scan is complete, construction of the brick's in-RAM key
catalog is finished.

See xref:checkpoints[] for details on brick checkpoint operations.

[[chain-repair]]
=== Chain Repair

When a chain is in the `degraded` state, new bricks that have entered
their `pre_init` state can become eligible to join the chain.  All
new bricks are added to the end of the chain and undergo the chain
repair process.

.Chain of length 2 in `degraded` state, a third brick under repair
svgimage::images/read-write-path-3-repair[align="center", scaledwidth="80%"]

The protocol used between upstream and downstream bricks is an
iterative protocol that has two phases in a single iteration.

1. The upstream brick sends a subset of {Key, Timestamp} tuples
downstream.
* The downstream brick deletes keys from its key catalog that do not
appear in the upstream's subset.
* The downstream brick replies with the list of keys that it does not
have or have older timestamps.
2. The upstream bricks sends full information (all key metadata and
value blobs) for all keys requested by the downstream in step #1.
* The downstream brick acknowledges the new/replacement keys.

When the repair is finished, the Admin Server will change the roles of
some/all chain members to make the repairing brick the new tail of the
chain.

Only one brick may be repaired at one time.  In theory it is possible
to repair multiple bricks simultaneously, but the extra code
complexity that would be required to do so has been judged to be too
expensive (so far).

==== Chain reordering when moving from `degraded` -> `healthy` states

[[chain-reordering-middle-brick-fails]]
.Chain order after a middle brick fails and is repaired (but not yet reordered)
svgimage::images/chain-fail-repair-reorder[align="center", scaledwidth="70%"]

After a middle brick fails and is repaired, the chain's ordering is:
brick 1 -> brick 3 -> brick 2.  According to the algorithm in the
original Chain Replication paper, the final chain ordering is
expected.  The Hibari implementation adds another step: reordering the
chain.

For chains longer than length 1, when the Admin Server moves the chain
from `degraded` -> `healthy` state, the Admin Server will reorder the
the chain to match the schema's definition for the healthy chain
order.  The assumption is that the Hibari administrator wishes the
chain use a very specific order when it is in the `healthy` state.
For example, if the chain's workload were extremely read-intensive,
the machine for logical brick #3 could have faster CPU or faster disks
than the other bricks in the chain.  To take full advantage of the
extra capacity, the chain should be reordered as soon as possible.

However, it is not easy to reorder the chain.  The replication of a
client update during the reordering could get lost and violate Hibari's
strong consistency guarantees.  The following algorithm is used to
preserve consistency:

1. Set all bricks to read-only mode.
2. Wait for all updates to sync to disk at each brick and to progress
   downstream fully from head -> tail.
3. Set brick roles to reflect the final desired order.
4. Set all bricks to read-write mode.
** Client `do` operations that contain updates will be resubmitted
   (via the client-side API function brick_server:do()) to the
   cluster.

Typically, executing this algorithm takes less than one second.
However, because the head brick is forced temporarily into read-only
mode, client update requests will be delayed until read-only mode is
turned off.

Client update requests submitted during read-only mode will be queued
by the head brick and will be processed when read-only mode is turned
off.  Client read-only requests are not affected by read-only mode.

// JWN: I think it might be helpful to mention/ to explain (but maybe
// not here) that Client updates may actually persist even though the
// client stopped waiting and returned a timeout to the "application".
// A Timeout on Client updates can not guarantee the change was
// applied or not applied to the Hibari tables.

[[checkpoints]]
=== Brick Checkpoint Operations

As updates are received by a brick, those updates are written to the
brick's private write-ahead log.  During normal operations, private
write-ahead log is write-only: the data there is only read at logical
brick initialization time.

The checkpoint operation is used to reclaim disk space in the brick's
private write-ahead log.  See xref:wal-dirs-and-files[] for a
description of log sequence files and xref:central-conf-parameters[]
for details on the `central.conf` configuration file.

.Brick checkpoint processing steps
1. When the total log size (i.e. total size of all log files in the
   brick's private log's shortterm storage area) reaches the size of the
   `brick_check_checkpoint_max_mb` parameter in `central.conf`, a
   checkpoint operation is started.
   * Assume that the current log sequence file number is *N*.
2. Two log sequence files are created, *N+1* and *N+2*.
3. Checkpoint data is written to log sequence number *N+1*.
4. New updates by clients and chain replication are written to log
   sequence number *N+2*.
5. Contents of the brick's in-RAM key catalog are dumped to log
   sequence file *N+1*, subject to the bandwidth constraint of the
   `brick_check_checkpoint_throttle_bytes` configuration parameter.
6. When the checkpoint is finished and flushed to disk, all log
   sequence files with a number less than or equal to *N* are deleted.

IMPORTANT: Each logical brick will checkpoint itself as its private
log grows.  It is possible that multiple logical bricks can schedule
checkpoint operations simultaneously.  The bandwidth limitation of the
`brick_check_checkpoint_throttle_bytes` parameter is applied to the
_sum of all writes by all checkpoint operations_.

[[scavenger]]
=== The Scavenger

As described in xref:write-ahead-logs[], all updates from all logical
bricks are first written to the ``common log''.  The most common of
these updates are:

* Metadata updates, e.g. key insert or key delete, by a logical brick.
* A new value blob associated with a metadata update such as a Hibari
client set operation.
** This type is only applicable if the brick is configured to store
value blobs on disk.  This configuration is defined (by default) on a
per-table basis and is then propagated to the chain and brick level by
the Admin Server.

As explained in xref:write-ahead-logs[], the write-ahead log provides
infinite storage at a logical level.  But in the physical level, disk
space must be reclaimed somehow.  Because the common log is shared by
multiple logical bricks, the technique described in xref:checkpoints[]
cannot be used by the common log.

A process called the ``scavenger'' is used to reclaim disk space in
the common log.  By default, the scavenger runs at 03:00 daily.  The
steps it executes are described below.

.Common log scavenger processing steps
1. For all bricks that store value blobs on disk, scan each logical
brick's in-RAM key catalog to create a list of all value blob storage
locations.
2. Sort the value blob location list by log sequence number.
3. Identify all log sequence files with a "live data ratio" of at
least *X* percent (default = 90%, see
`brick_skip_live_percentage_greater_than` configuration parameter).
4. For all log files where live data ratio is less than *X*%, copy
value blobs to new log sequence files.  This copying is limited by the
amount of bandwidth configured by `brick_scavenger_throttle_bytes` in
`central.conf`.
5. When all blobs have been copied out of an old log sequence file and
flushed to stable storage, update the storage locations in the in-RAM
key catalog, then delete the old log sequence file.

ifdef::theme[]
image:images/scavenger-techpubs.png[]
endif::theme[]
ifndef::theme[]
image:images/scavenger-techpubs.png[width="65%"]
endif::theme[]

IMPORTANT: The value of the `brick_skip_live_percentage_greater_than`
configuration parameter determines how much additional disk space is
required to store *X* gigabytes of live data.  If the parameter is
*N*, then 100-*N* percent of all common log disk space may be wasted
by storing dead data.

IMPORTANT: Additional disk space is required to log all updates that
are made after the scavenger has run.  This includes space in the
common log as well as in each logical brick private logs (subject to
the general limit of the `brick_check_checkpoint_max_mb` configuration
parameter.

IMPORTANT: The current implementation of Hibari requires that
plenty of disk space _always_ be available for write-ahead logs and
for scavenger operations.  We strongly recommend that the
`brick_scavenger_temp_dir` configuration item use a different file
system than the `application_data_dir` parameter.  This directory
stores temporary files required for sorting and other operations that
would otherwise require large amounts of RAM.
