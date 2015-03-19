Backup and Disaster Recovery
============================

=== Backup and Recovery Software

At the time of writing, Hibari's largest cluster deployment is:

* Well over 50 physical bricks
* Well over 4TB of disk space per physical brick
* Single data center, operated by a telecom carrier and integrated
  with third-party monitoring and control software

If a backup were made of all data in the cluster, the biggest question
is, "Where would you store the backup?"  Given the cluster's purpose
(real-time email/messaging services), the quality of the data center's
physical and software infrastructures, the length of the Hibari chains
used for physical data redundancy, the business factors influencing
the choice not to deploy a "hot backup" data center, and other
factors, Cloudian has not developed the backup and recovery software for
Hibari. Cloudian's smaller Hibari deployments also resemble the largest
deployment.

However, we expect that backup and recovery software will be high
priorities for open source Hibari users.  Together with the open
source users and developers, we expect this software to be developed
relatively quickly.

=== Disaster Recovery via Remote Data Centers

==== Single Hibari cluster spanning two data centers

It is certainly possible to deploy a single Hibari cluster across two
(or more) data centers.  At the moment, however, there is only one way
of doing it: each chain of data replication must have a brick located
in each data center.

As a consequence of brick placement, it is mandatory that Hibari
clients pay the full round-trip latency penalty for each update.  See
xref:diagram-write-path-3[] for a diagram; the "head" and "tail"
bricks would be in separate data centers, using WAN network
connectivity between them.

For some applications, strong consistency is a higher priority than
low latency (both for writes and possibly for reads, if the client is
not co-located in the same data center as the chain's tail brick).  In
those cases, such cross-data-center brick placement can make sense.

However, Hibari's Admin Server cannot handle all failure scenarios,
especially when WAN connectivity is broken between data centers; more
programming work is required for the Admin Server to automate the
handling of all processes.  Furthermore, Hibari's basic design cannot
tolerate network partitions well, see xref:cap-theorem-and-hibari[]
and xref:admin-server-and-network-partition[].  If the Admin Server
were capable of handling WAN network partitions, it's almost certain
that all Hibari nodes in one of the partitioned data centers would be
inactive.

==== Multiple Hibari clusters, one per data center

Conceptually, it's possible to run multiple Hibari clusters, one per
data center.  However, Hibari does not have the software required for
WAN-scale replication.

In theory, such software isn't too difficult to develop.  The tail
brick of each chain can maintain a log of recent updates to the
chain.  Those updates can be transmitted asynchronously across a WAN
to another Hibari cluster in a remote data center.  Such a scheme is
depicted in the figure below.

[[async-replication-try1]]
.A future scenario of asynchronous, cross-data-center Hibari replication
svgimage::images/async-replication-try1[align="center", scaledwidth="80%"]

This kind of replication makes the most sense if "Data Center #1"
were in an active role and Data Center #2" were in a hot-standby
role.  In that case, there would never be a "Data Center #2 Client",
so there would be no problem of strong consistency violations by
clients accessing both Hibari clusters simultaneously.  The only
consistency problem would be one of durability: the replay of async
update logs every `N` seconds would mean that up to `N` seconds of
updates within "Data Center #1" could be lost.

However, if clients access both Hibari clusters simultaneously, then
Hibari's strong consistency guarantee would be violated.  Some
applications can tolerate weakened consistency.  Other applications,
however, cannot.  For the those apps that must have strong
consistency, Hibari will require additional design and code.

TIP: A keen-eyed reader will notice that xref:async-replication-try1[]
is not fully symmetric.  If clients in "Data Center #2" make updates
to the chain, then the same async update log maintenance and replay to
"Data Center #1" would also be necessary.
