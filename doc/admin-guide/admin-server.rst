The Admin Server Application
============================

The Hibari ``Admin Server'' is an OTP application that runs in an
active/standby configuration within a Hibari cluster.  The Admin
Server is responsible for:

* Monitoring the health of each brick in the cluster,
  see xref:brick-lifecycle-fsm[].
* Monitoring the status of each chain in the cluster,
  see xref:chain-lifecycle-fsm[].
* Managing administrative changes of chain -> brick mappings,
  see xref:chain-length-change[].
* Managing data rebalancing, see xref:chain-migration[].
* Communicating cluster status to Hibari client nodes.
* Other administrative tasks, such as the creation of new tables.

Only one instance of the Admin Server is permitted to run within the
cluster at a time.  The Admin Server runs in an ``active/standby''
configuration that is used in many high-availability clustered
applications.  The nodes that are eligible to participate in the
active/standby configuration are configured via the main Hibari
configuration file; see xref:admin-server-in-central-conf[] and
xref:central-conf-parameters[] for more details.

=== Admin Server Active/Standby Implementation

The active/standby application failover is handled by the Erlang/OTP
application controller.  No extra third-party software is required.
See Chapter 7, "Applications", and Chapter 9, "Distributed
Applications", in the "OTP Design Principles User's Guide" at
http://www.erlang.org/doc/design_principles/distributed_applications.html.

[[bootstrap-bricks]]
=== Admin Server's Private State: the Bootstrap Bricks

On each active and standby node, there is a hint file called
`Schema.local` which contains the name of the ``bootstrap bricks''.
These bricks operate outside of the chain replication algorithm to
provide redundant, persistent state for the Admin Server application.
See xref:bricks-outside-chain-replication[] for a short summary of
standalone bricks.

All of the Admin Server's private state is stored in the bootstrap
bricks.  This includes:

* All table definitions and their configuration, e.g. consistent
  hashing parameters.
* Status of all bricks and all chains.
* Operational history of all bricks and all chains.

With the help of the Erlang/OTP application controller and the Hibari
Partition Detector application, only a single instance of the Admin
Server is permitted to run at any one time.  That single application
instance has full control over the data stored in the bootstrap bricks
and therefore does not have to manage concurrent updates to bootstrap
brick data.

=== Admin Server Crash and Restart

When the Admin Server application is stopped (e.g. node shutdown) or
crashes (e.g. software bug, power failure), all of the tasks outlined
at the beginning of xref:admin-server-app[] are halted.  In theory,
the 20-30 seconds that are required for the Admin Server to restart
could mean 20-30 seconds of negative service impact to Hibari clients.

In practice, however, Hibari clients almost never notice when an Admin
Server instance crashes and restarts.  Hibari clients do not need the
Admin Server when the cluster is stable.  The Admin Server is only
necessary when the state of the cluster changes.  Furthermore, as far
as clients are concerned, clients are only affected when bricks crash.
Other cluster change events, such as when chain replication repair
finished, do not directly impact clients and thus can wait for the
Admin Server to finish restarting.

A Hibari client will only notice an Admin Server crash if another
logical brick crashes while the Admin Server is temporarily out of
service.  The reason is due to the nature of the Admin Server's
responsibilities.  When chain is broken by a brick failure, the
remaining bricks must have their roles reconfigured to put the chain
back into full service.  The Admin Server is the only automated entity
that is permitted to change the role of a brick.  For more details,
see:

* xref:brick-lifecycle-fsm[]
* xref:chain-lifecycle-fsm[], and
* xref:chain-repair[].

[[admin-server-and-network-partition]]
=== Admin Server and Network Partition

One feature of the Erlang/OTP application controller is that it is not
robust in event of a network partition.  To prevent multiple Admin
Server apps running simultaneously, another application is bundled
with Hibari: the Partition Detector.  See xref:partition-detector[]
for an overview and explanation of the 'A' and 'B' physical networks.

As described briefly in xref:cap-theorem-and-hibari[], Hibari does
support the "Partition tolerance" aspect of Eric Brewer's CAP theorem.
More specifically, if a network partition occurs, and a Hibari cluster
is split into two or more pieces, not all clients on both/all sides of
the network partition will be able to access Hibari services.

For the sake of discussion, we assume the cluster has been split into
two fragments by a single partition, though any number of fragments may
happen in real use.  We also assume that nodes on both sides of the
partition are configured in standby roles for the Admin Server.

If a network partition event happens, the following events will soon
follow:

* The OTP application controller for some/all
  `central.conf`-configured nodes will notice that communication with
  the formerly active Admin Server is now impossible.
* Using internal logic, each application controller will make a
  decision of which standby node should move to active status.
* Each active status node will start an instance of the Admin Server.

Note that all steps above will happen in parallel on nodes on both
sides of the partition.  If this situation is permitted to continue,
the invariant of "Admin Server may only run on one node at a time"
will be violated.  However, with the help of the Partition Detector
application, multiple Admin Server instances can be detected and
halted.

UDP broadcasts on the 'A' and 'B' networks can help the Admin Server
determine if it was restarted due to an Admin Server crash or by a
network partition.  In case of a network partition on network 'A', the
broadcasts on network 'B' can indicate that another Admin Server
process remains alive.

If multiple Admin Server instances are detected, the following logic
is used:

* If an Admin Server is in its "running" phase, then any other any
  Admin Server instance that is still in its "initialization" phase
  will halt.
* If multiple Admin Server instances are all in the "initialization"
  phase, then only the Admin Server instance with the smallest name
  (in lexicographic sorting order) is permitted to run: all other
  instances will halt.

==== Importance of two physically separate networks

IMPORTANT: It is possible for both the 'A' and 'B' networks to
partition simultaneously.  The Admin Server and Partition Detector
applications cannot always correctly react to such events.  It is
extremely important that the 'A' and 'B' networks be separate physical
networks, including: separate physical network interfaces on each
brick, separate cabling, separate network switches, and all other
network-related equipment also be physically separate.

It is possible to reduce the reliance on multiple physical networks
and the Partition Detector application, but such techniques have not
been added to Hibari yet.  Until an alternative network partition
mitigation mechanism is implemented, we strongly recommend the proper
configuration of the Partition Detector app and all of its hardware
requirements.

=== Admin Server, Network Partition, and Client Access

When a network partition event occurs, there are two cases that affect
a client's ability to work with the cluster.

* The client machine is on the same side of the partition as the Admin
  Server.
* The client machine is on the opposite side of the partition as the
  Admin Server.

If the client machine is on the same side of the partition, the client
may see no interruption of service at all.  If the Admin Server is
restarted in reaction to the partition event, there may be a small
window of time (e.g. 20-30 seconds) where requests might fail because
the Admin Server has not yet reconfigured chains on this side of the
partition.

If the client machine is on the opposite side of the partition, then
the client will not have access to the Admin Server and may not have
access to properly configured chains.  If a chain lies entirely
entirely on the same side of the partition as the client, then the
client can continue to use that chain successfully.  However, any
chain that is "cut in two" by the partition cannot support updates by
any client.
