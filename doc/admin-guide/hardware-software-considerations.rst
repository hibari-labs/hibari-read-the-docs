Hardware and Software Considerations
====================================

As noted in xref:hibari-origins[], at the time of writing, Hibari has
been deployed exclusively in data centers run by telecom carriers.
All carriers have very specific requirements for integrating with its
existing deployment, network monitoring, alarm management, and other
infrastructures.  As a result, many of those features have been
omitted to date from Hibari.  With Hibari's release into an "open
source environment", we expect that these gaps will be closed.

Hibari's carrier-centric heritage has also influenced the types of
hardware, networking gear, operating system, support software, and
internal Hibari configuration that have been used successfully to
date.  Some of these practices will change as Hibari evolves from its
original use patterns.  Until then, this section discusses some of the
things that a systems/network administrator must consider when
deploying a Hibari cluster.

Similarly, application developers must be very familiar with these
same issues.  An unaware developer can create an application that uses
too many resources on under-specified hardware, causing problems for
developers, support staff, and application users alike.  We wish
Hibari to grow and flourish in its non-relational DB niche.

[[brick-hardware]]
=== Notes on Brick Hardware

==== Lots of RAM is better

Each Hibari logical brick stores all information about its keys in
RAM.  Both the logical brick's private write-ahead log and the common
write-ahead log are not "disk-based data structures" in the typical
sense, such as on-disk hash tables or B-trees.  Therefore, Hibari
bricks require a lot of RAM to function.

For more details, see:

* xref:overview-high-performance[]
* xref:per-table-config-perf-options[] ... if a table stores its value
  blobs in RAM, it will consume more RAM than if those value blobs are
  stored on disk.
* xref:hibari-data-model[]
* xref:brick-init[]

==== Lots of disk I/O capacity is better

By default, Hibari will write and flush each update to disk before
sending a reply downstream or back to the client.  Hibari will perform
better on systems that have higher disk I/O capacity.

* Non-volatile/battery-backed cache on the disk controller(s) is
  helpful, when combined with a write-back cache policy.  The more
  cache, the better.  If the read/write ratio of the cache can be
  changed, a default policy of 10/90 or 0/100 (i.e. skewed to writes)
  is typically more helpful than a default 50/50 split.
* On-disk (volatile) cache on individual disks is not helpful.
* Faster spinning disks are more helpful than slower spinning disks.
* If using RAID, a large stripe width of e.g. 512KBytes or 1024KBytes
  is usually more helpful than the (usually) smaller default stripe
  width on most controllers.
* If using RAID, a hardware RAID implementation may be very slightly
  helpful.
* RAID redundancy (e.g. RAID 1, 10, 5, 6) is not required by Hibari,
  but it can help reduce the odds of failure of an individual physical
  brick.  If physical bricks do not use data redundant RAID
  (e.g. RAID 0, concatenation), it's a good idea to consider using
  longer replication chains to compensate.

For more details, see:

* xref:the-physical-brick[]
* xref:per-table-config-perf-options[]
* xref:hibari-data-model[]

[[high-io-rate-devices]]
==== High I/O rate devices (e.g. SSD) may be used

Hibari has some support for high I/O rate devices such as solid state
disks, flash memory disks, flash memory storage cards, et al.  There
is nothing in Hibari's implementation that would preclude using
high-speed disk devices as the only storage for Hibari write-ahead
logs.

Hibari has a feature that can segregate high write I/O with `fsync(2)`
operations onto a separate high-speed device, and use cheaper &
lower-speed Winchester disk devices for bulk storage.  This feature
has not yet been well-tested and optimized.

For more details, see:

* xref:write-ahead-logs[]
* xref:two-wal-types[]

==== Lots of disk storage capacity may be a secondary concern

More disks of smaller capacity are almost always more helpful than a
few disks of larger capacity.  RAID 0 (no data redundancy) or RAID 10
("mirror" data redundancy) is useful for combining the I/O capacity of
multiple disks into a single logical volume.  Other RAID levels, such
as 5 or 6, can be used, though at the expense of higher write I/O
overhead.

For more details, see:

* xref:write-ahead-logs[]

[[considerations-cpu]]
==== Lots of CPU capacity is a secondary concern

Hibari storage bricks do not, as a general rule, require large amounts
of CPU capacity.  The largest single source of CPU consumption is in
MD5 checksum calculation.  If the data objects most commonly written &
read by your application are small, then multi-socket, multi-core CPUs
are not required.

Each Hibari logical brick is implemented within the Erlang virtual
machine as a single `gen_server` process.  Therefore, each logical
brick can (generally speaking) only fully utilize one CPU core.  If
your Hibari cluster appears to have CPU-utilization imbalance, then
the recommended strategy is to change the chain placement policy of
the chains.  For example, there are two methods for arranging a chain
of length three across three physical bricks:

[[1-chain-striped-across-3-bricks]]
The first example shows one chain striped across three physical
bricks.  If the read/write ratio for the chain is extremely high
(i.e. most operations are reads), then most of the CPU activity (and
perhaps disk I/O, if blobs are stored on disk) will be directed to the
"Chain 1 tail" brick and cause a CPU utilization imbalance.

.One chain striped across three physical bricks
  | Physical Brick X | Physical Brick Y | Physical Brick Z |
  ----------------------------------------------------------
     Chain 1 head   ->  Chain 1 middle ->  Chain 1 tail

[[3-chains-striped-across-3-bricks]]
The second example shows the same three physical bricks but with three
chains striped across them.  In this example, each physical brick is
responsible for three different roles: head, middle, and tail.
Regardless of the read/write operation ratio, all bricks will utilize
roughly the same amount of CPU.

.Three chains striped across three physical bricks
  | Physical Brick T | Physical Brick U | Physical Brick V |
  ----------------------------------------------------------
     Chain 1 head   ->  Chain 1 middle ->  Chain 1 tail   ||
     Chain 2 tail   ||  Chain 2 head   ->  Chain 2 middle ->
     Chain 3 middle ->  Chain 3 tail   ||  Chain 3 head   ->

In multi-CPU and multi-core systems, a side-effect of using more
chains (and therefore more bricks) is that the Erlang virtual machine
can schedule more logical brick computation across a larger number of
cores and CPUs.

=== Notes on Networking

Hibari works quite well using commodity "Gigabit Ethernet" interfaces.
Lower latency (and higher cost) networking gear, such as Infiniband,
is not required.

For production use, it is _strongly recommended_ that all Hibari
servers be configured with two physical network interfaces, cabling,
switches, etc.  For more details, see:

* xref:partition-detector[]

==== Client protocol load balancing

The native Erlang client, via the `gdss` or `gdss_client` OTP
applications, do not require any load balancing.  The Erlang client
already is a participant in the consistent hashing algorithm (see
xref:consistent-hashing-example[]).  The Admin Server distributes
updates to a table's consistent hash map each time cluster membership
or chain/brick status changes.

All other client access protocols are "dumb", by comparison.  Take for
example the Amazon S3 protocol service.  There is no easy way for a
Hibari cluster to convey to a generic HTTP client how to calculate
which brick to send a query to.  The HTTP redirect mechanism could be
used for this purpose, but other protocols don't have an equivalent
feature.  Also, the latency overhead of sending a redirect is far
higher than Hibari's solution to this problem.

Hibari's solution is simple: the Hibari server-side "dumb" protocol
handler uses the same native Erlang client that any other Hibari
client app written in Erlang users.  That client is capable of making
direct routing decisions.  Therefore, the "dumb" protocol handler
within a Hibari node acts as a translating proxy: it uses the "dumb"
client access protocol on one side and uses the native Erlang client
API on the other.

.Hibari "dumb" protocol proxy
svgimage::images/dumb-protocol-proxy[align="center", scaledwidth="80%"]

The deployed "state of the art" for such dumb protocols is to use a
TCP load balancer (aka a "layer 4" load balancer) to spread dumb
client workload across multiple Hibari dumb protocol servers.

=== Notes on Operating System

Hibari servers operate on top of the Erlang virtual machine.  In
principle, any operating system that is supported by the Erlang
virtual machine can support Hibari.

==== Supported Operating Systems

In practice, Hibari is supported on the following operating systems:

* Linux x86_64
  ** Red Hat Enterprise Linux 5.x and 6.x (RHEL 5.3 is used in
     production and QA environments within Cloudian, Inc.)
  ** CentOS 5.x and 6.x
  ** Ubuntu 12.04 LTS or newer
* Linux ARMv7 (32 bit)
  ** Ubuntu 12.04 LTS or newer
  ** Hibari runs on Calxeda EnergyCore based super high-density,
     scale-out cluster
* Unix Solaris variants
  ** Joyent SmartOS (64 bit)
* Mac OS X
* FreeBSD (though not currently in a jail environment, due to some TCP
  services getting EPROTONOSUPPORT errors)

The versions recently tested for Hibari by the community:

- CentOS 6.3 (x86_64)
- Ubuntu 12.04 LTS (ARMv7)
- Joyent SmartOS 20130221 (64 bit)

To take advantage of RAM larger than 4GB, we recommend that you use
a 64-bit version of your OS's kernel, 64-bit versions of the user
runtime, and a 64-bit version of the Erlang/OTP runtime.

[[os-readahead-configuration]]
==== OS Readahead Configuration

Some operating systems have support for OS-based "readahead":
pre-fetching blocks of a file with the expectation that those blocks
will soon be requested by the application.  Properly configured,
readahead can substantially raise throughput and reduce latency on
many read-heavy I/O workloads.

The read I/O workloads for Hibari fall into two major categories:

1. Extremely predictable sequential read-only I/O during brick
   initialization (see xref:brick-init[]).
2. Extremely unpredictable random read I/O for fetching value blobs
   from disk.

The first I/O pattern can usually benefit a great deal from an aggressive
readahead policy.  However, an aggressive readahead policy can have
the opposite effect on the second I/O pattern.  Readahead policies
under Linux, for example, are defined on a per-block device basis and
does not change in response to application runtime behavior.

If your OS supports readahead policy configuration, we recommend using
a small read and then measuring its effect with a real or simulated
workload with the real Hibari server.

[[disk-scheduler-configuration]]
==== Disk Scheduler Configuration

We recommend that you experiment with disk scheduler configuration on
relevant OSes such as Linux.  The "deadline" scheduler is likely to
provide better performance characteristics.

=== Notes on Supporting Software

A typical "server" type installation of a Linux or FreeBSD OS is
sufficient for Hibari.  The following is an incomplete list of other
software packages that are necessary for Hibari's installation and/or
runtime.

* NTP
* Erlang/OTP version R13B04
* Either "lynx" or "elinks", a text-based Web browser

// JWN: This seems like a good place to mention patches that are
// needed beyond R13B04 ... busy dist port?

[[ntp-config-strongly-recommended]]
==== NTP configuration of all Hibari server and client nodes

It is strongly recommended that all Hibari server and client nodes
have the NTP daemon (Network Time Protocol) installed, properly
configured, and running.

* The `brick_simple` client API uses the OS clock for automatic
  generation of timestamps for each key update.  The application
  problems caused by badly out-of-sync OS clocks can be easily avoided
  by NTP.
* If a client's clock is skewed by more than the
  `brick_do_op_too_old_timeout` configuration attribute in
  `central.conf` (units = milliseconds), then the brick will silently
  discard the client's operation.  The only symptoms of this are:
** Client-side timeouts when using the `brick_simple`, `brick_server`,
  or `brick_squorum` APIs.
** Increasing `n_too_old` statistic counter on the brick.

=== Notes on Hibari Configuration

There are several reasons why disk I/O rates can temporarily increase
within a Hibari physical brick:

* Logical brick checkpoints for increased write I/O ops, see
  xref:checkpoints[]
* The common log "scavenger" for increased read and write I/O ops,
  see xref:scavenger[]
* Chain replication repair, see xref:chain-repair[]
** As the upstream/"repairer" brick, the extra read I/O ops,
   if the brick stores value blobs on disk
** As the downstream/"repairee" brick, extra write I/O ops

The Hibari `central.conf` file contains parameters that can limit the
amount of disk bandwidth used by most of these operations.

See also:

* xref:considerations-cpu[]
* xref:central-conf-parameters[]

=== Notes on Monitoring a Hibari Cluster

The Admin Server's status page contains current status information
regarding all tables, chains, and bricks in the cluster.  By default,
this service listens to TCP port 23080 and is reachable via HTTP at
http://any-hibari-node-name:23080/.  HTTP redirect will steer your
browser to the Admin Server node.

* Hypertext links for each table, chain, and brick can show more
  detailed info on each entity.
* The "Dump History" link at the bottom of the Admin Server's HTTP
  status page can show operations history across multiple bricks,
  chains, and/or tables by using the regular expression feature.
* Each logical brick maintains counters of each type of Hibari client
  op primitive.  At present, these stats are only exposed via the HTTP
  status server or by the native Erlang interface, but it's possible
  to expose these stats via SNMP and other protocols in a
  straightforward manner.
** Stats include: number of `add`, `replace`, `set`, `get`,
  `get_many`, `delete`, and micro-transactions.

==== Hibari Admin Server HTTP status

For example screen shots of the Admin Server status pages (a work in
progress), see link:./misc-screenshots/admin-server-status/index.html[].

See also:

* xref:chain-lifecycle-fsm[]
* xref:brick-lifecycle-fsm[]
