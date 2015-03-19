Why Hibari?
-----------

Hibari was developed internally by Cloudian, Inc. (formerly Gemini
Mobile Technologies), a leading producer of mass-scale messaging and
transaction systems for Tier 1 mobile operators in Asia, Europe, and
the Americas. Cloudian had need for a data store that was efficient,
fast, flexible, and scalable, as well as robust enough to withstand
the rigors of deployment in Tier 1 telecom production environments.
Dissatisfied with the then-available options, Cloudian in 2005 began
work on what came to be Hibari (the name is Japanese for skylark; the
kanji characters stand for "cloud bird").

With the system having in recent years matured and been proven in
production, Cloudian released Hibari to the open source community in
July 2010 under the Apache 2.0 license. Cloudian regards the open
source community as the best venue in which Hibari can continue to
perfect and grow.

This section describes some of the distinctive features that make
Hibari a very attractive option for businesses and developers seeking
a modern Big Data storage system:

- link:#engineered-erlang[Engineered in Erlang]
- link:#chain-replication[Chain Replication for High Availability and Strong Consistency]
- link:#scalability[Easy, Affordable Scalability]
- link:#high-performance[High Performance, Especially for Reads and Large Values]
- link:#simple-powerful-api[Simple But Powerful Client API]
- link:#production-proven[Production-Proven]
- link:#hibari-benefits-by-user[Hibari Benefits for Developers, System Administrators, and Businesses]

[[engineered-erlang]]

Engineered in Erlang
^^^^^^^^^^^^^^^^^^^^

Erlang is a general purpose programming language and runtime
environment designed specifically to support reliable,
high-performance distributed systems. Originally developed by Ericsson
in the 1980s for building advanced telecom networking systems,
Erlang/OTP (Open Telecom Platform) was open-sourced in 1998. Hibari is
written entirely in Erlang.

Erlang provides a range of benefits that make it the ideal foundation
for a distributed key-value storage solution:

- **Concurrency**. Erlang has extremely lightweight processes that
  communicate by message passing and have no shared memory.
  Scheduling, memory management, and other concurrency-related
  services are managed by the Erlang VM, placing  no requirements for
  concurrency on the host operating system.

- **Distribution**. Erlang is designed specifically for distributed
  environments. Passing messages transparently via TCP, Erlang
  processes on different nodes communicate with each other in exactly
  the same way as do processes on the same node. The simple and
  efficient design facilitates massive parallelism and scalability of
  the sort required by a high-performance distributed storage
  system. With its prowess for concurrency and distributed
  processing, it has been suggested that Erlang can be regarded as a
  first-of-its-kind
  http://www.oreillygmt.eu/open-sourcefree-software/erlang-the-ceos-view/["application
  system"], analogous to an operating system except running across
  and coordinating multiple hosts.

- **Robustness**. Erlang processes are completely independent of each
  other, with no data sharing. While functionally isolated, Erlang
  processes are able to monitor each other and to detect and respond
  to crashed processes, even on remote nodes.

- **Portability**. The same Erlang VM can run on Linux, Unix, Windows,
  Macintosh, or VxWorks. Distributed Erlang processes can seamlessly
  communicate with each other regardless of the heterogeneity of
  their host operating systems. This OS portability is a valuable
  facilitator of storage system elasticity, as system managers may
  need to mix and match hosts in response to fluid demand
  environments.

- **Hot code upgrades**. Erlang-based applications like Hibari support
  hot code upgrades: upgrades can be applied without shutting down
  the system. During the change-over, old and new code can run
  simultaneously. This is a key benefit for environments that require
  "always-on" availability for end users.

Other features reinforce Erlang's suitability for reliable distributed
applications, including incremental garbage collection,
single-assignment variables, and robust exception handling.


[[chain-replication]]

Chain Replication for High Availability and Strong Consistency
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Hibari distributed key-value store implements a version of the
chain replication methodology first proposed by
http://www.usenix.org/event/osdi04/tech/full_papers/renesse/renesse.pdf[van
Renesse and Schneider] to achieve redundancy and high availability
without sacrificing data consistency. At a high level, chain
replication in a Hibari storage cluster works as follows:

- Through consistent hashing, the key space is divided across multiple
  storage "chains".
- Each chain is composed of multiple logical storage "bricks", with
  each brick running in its own Erlang VM instance.
- Within each chain, the member bricks have differentiated
  roles. Client-requested updates to key-value pairs are written first
  to the "head" brick, then automatically replicated downstream to one
  or more "middle" bricks and finally to the "tail" brick, which
  returns an update acknowledgement to the client. By contrast, read
  requests are directed to the tail brick, which returns the response
  to the client.

image:images/chain_replication.png[]

While most distributed storage systems are able to guarantee only weak
or eventual data consistency across replicas -- placing the burden on
the client application (and the client application developer) to
manage the potential inconsistencies -- Hibari with its chain
replication implementation guarantees strong consistency. Data updates
are considered complete, and are acknowledged to clients, only when
they have replicated through the chain to the tail; and read requests
are processed only by the tail. Consequently, after an object update
is acknowledged to a Hibari client, other clients are guaranteed to
see only the newest version of that object. This strong consistency is
valuable in environments where 'eventual consistency' is at odds with
the service level expected by end users, or where system designers do
not want to clutter client applications with the logic required to
manage data inconsistency.

The "length" of a chain is configurable and can be based on your
desired degree of replication and redundancy. For example, a chain of
length four would have a head brick, two middle bricks, and a tail
brick; while a three-brick chain would have a head, one middle, and a
tail. A chain can also operate at length two (a head and tail, with no
middle) and even at length one (one brick playing both the head role
and the tail role).

Because chains can operate at any length, and because the system is
able to detect failures within the chain and to adjust member brick
roles accordingly, Hibari delivers high availability as well as strong
data consistency. For example, if in a three-brick chain the head
brick goes down, the middle brick automatically takes over the head
brick role, allowing the chain to continue functioning normally:

image:images/automatic_failover.png[]

If the new head brick failed also, the lone remaining brick would then
play both the head role and the tail role, processing all writes and
reads itself as a single-brick "chain".

While multiple logical bricks can run on a single physical node, for
high availability it is of course desirable that a particular chain's
member bricks be deployed on separate machines. If you want to run
multiple bricks per machine and also ensure high availability for each
chain, an attractive deployment option is to "stripe" the chains
across machines:

image:images/load_balanced_chains.png[]

Note also that because head bricks (receiving incoming write requests)
and tail bricks (replying to write requests and processing read
requests) bear more load than do middle bricks, load balancing across
machines can be achieved in part by allocating the different brick
roles evenly, as in the diagram above.

In the event of a physical node failure, bricks within each impacted
chain automatically shift roles, and each chain continues to provide
normal service to clients:

image:images/automatic_failover_2.png[]

For further information about chain replication, fail-over, and
recovery in a Hibari storage system, and for information about
Hibari's redundantly structured cluster membership application called
the Admin Server, see these sections of the Hibari System
Administrator's Guide:

- link:hibari-sysadmin-guide.en.html#hibari-architecture[Hibari Architecture]
- link:hibari-sysadmin-guide.en.html#life-of-brick[The Life of a (Logical) Brick]
- link:hibari-sysadmin-guide.en.html#dynamic-cluster-reconfiguration[Dynamic Cluster Reconfiguration]
- link:hibari-sysadmin-guide.en.html#admin-server-app[The Admin Server Application]

[[scalability]]

Easy, Affordable Scalability
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Hibari provides Big Data scalability while minimizing the cost and
operational complexity of cluster growth:

- Hibari scales horizontally by the addition of more chains, deployed
  on more physical nodes. The total storage and processing capacity of
  a Hibari cluster increases linearly as machines are added to the
  cluster.
- The system rebalances data storage distribution automatically as
  chains are added to (or removed from) the cluster, with no
  downtime. You can grow (or shrink) your Hibari storage cluster with
  no service interruption.
- Hibari runs on commodity PCs. Further, the system easily
  accommodates heterogeneous hardware resources. Bricks within the
  storage cluster can have different RAM and disk sizes, and different
  CPU speeds. You can tune Hibari's consistent hash function to
  optimize your cluster's utilization of mixed hardware. Each chain
  can be assigned a weighting factor that will increase or decrease
  that chain's portion of the overall key space, relative to other
  chains.

In addition to supporting mixed hardware, Erlang-based Hibari can run
on most any OS. With its easy adaptability to disparate hardware and
operating systems, you can scale Hibari incrementally, with whatever
resources you have available. It's not necessary to buy all your
resources at once, or all of the same kind.

.. note::
   The outer limits of Hibari's horizontal scalability have not been
   definitely determined, but 200 to 250 nodes is a practical boundary
   due to the limitations of Erlang's built-in network distribution
   implementation. Also, while Hibari chains could theoretically be
   stretched across multiple data centers to provide geographic
   redundancy, to date only single data center deployments have been
   tested and used in production.

For further information on resizing a Hibari cluster, see
link:hibari-sysadmin-guide.en.html#dynamic-cluster-reconfiguration[Dynamic
Cluster Reconfiguration] in the Hibari System Administrator's Guide.

[[high-performance]]

High Performance, Especially for Reads and Large Values
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Several features work in combination to drive high performance in a
Hibari storage cluster, even at Big Data scale:

- The Erlang technology that underlies Hibari was specifically
  designed for and excels at distributed parallel processing.
- Hibari's implementation of consistent hashing and chain replication
  partitions the key-space across multiple chains, enabling parallel
  simultaneous processing of requests incoming to individual
  chains. The distribution of data across chains is tunable to allow
  optimal utilization of heterogeneous hardware resources.
- Hibari's chain replication implementation further aids performance
  by assigning storage bricks differentiated processing roles as head,
  middle, or tail. This division of labor particularly benefits read
  performance, as read requests are processed by "tail" bricks that do
  not bear the load of initial processing of write requests (since
  that work is done by "head" bricks).
- Hibari supports a number of performance-tuning options on a
  per-table basis. For example, while some distributed KVDBs support
  only disk-based storage or only RAM-based storage of value blobs,
  Hibari lets you choose RAM+disk-based or disk-only storage on a
  per-table basis, depending on your application needs. Whichever
  storage option you choose, all data changes are logged to disk to
  ensure data durability in the event of power failures. A batch
  commit technique is used to minimize disk I/O.

Leveraging this feature set, Hibari is able to deliver scalable high
performance that is competitive with leading open source NOSQL storage
systems, while also providing the data durability and strong
consistency that many systems lack. Hibari's performance relative to
other NOSQL systems is particularly strong for reads and for large
value (> 200KB) operations. Hibari's consistently high performance
even for large values distinguishes the system from solutions that are
tailored toward small value operations.

As one example of real-world performance, in a multi-million user
webmail deployment equipped with traditional HDDs (non SSDs), Hibari
is processing about 2,200 transactions per second, with read latencies
averaging between 1 and 20 milliseconds and write latencies averaging
between 20 and 80 milliseconds.

[[simple-powerful-api]]

Simple But Powerful Client API
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As a key-value store, Hibari's core data model and client API model
are simple by design: blob-based key-value pairs can be inserted,
retrieved, and deleted from lexicographically sorted tables. While
Hibari thus provides the flexibility and scalability associated with
key-value stores, the system also provides distinctive features that
enhance the power of client applications and developers:

- Clients can optionally assign per-object expiration times.
- Clients can optionally assign per-object custom flags. This
  flexible, custom meta-data can be updated with or without updating
  the associated value blob, and can be retrieved with or without the
  value blob.
- Objects are automatically timestamped each time they are
  updated. This timestamping mechanism facilitates "test-and-set" type
  operations: clients can specify that a requested operation be
  performed only if the target key's timestamp matches the client's
  expectations.
- Within key-prefix range limits (specifically, within individual
  chains but not across chains), Hibari's client API supports atomic
  transactions. This support for "micro-transactions" sets Hibari
  apart from other open source KVDBs and can greatly simplify the
  creation of robust client applications.

Hibari supports multiple client API implementations including:

- Native Erlang
- Universal Binary Format (UBF)
- Thrift
- Amazon S3
- JSON-RPC

You can develop Hibari client applications in a variety of languages
including Java, C/C++, Python, Ruby, and Erlang.

For further information about Hibari's client API, see
link:#client-api-erlang[Client API: Native Erlang] and the subsequent
client API chapters in this guide.

.. note::
   The Hibari source distribution does not include Amazon S3 and
   JSON-RPC. They are separate external projects.


[[production-proven]]

Production-Proven
^^^^^^^^^^^^^^^^^

While initial development work on Hibari was geared generally toward
the data storage demands of the Tier 1 telecom sector, as the system
evolved it needed to meet the requirements of a particular major Asian
carrier that wished to launch a GB webmail service. This customer's
requirements for Hibari included the following:

- Several million users from the start.
- Several billion stored messages within a few months of launch.
- Hundreds of TB storage capacity.
- Elasticity to support continual growth.
- Low system costs, particularly since the service would employ the
  "freemium" model.
- Individual messages could range in size from a few bytes to many MB
  with attachments.
- Support for per-object meta-data required.
- Strong consistency required, for interactive sessions.
- Data durability required -- loss of messages or meta-data unacceptable.
- High availability -- an "always on", branded service.
- Low latency, with < 1 second response times for end user transactions.

Hibari was built to meet these rigorous requirements, was hardened
through extensive testing and trials, and went live in support of this
large-scale webmail system at the beginning of 2010. The system now
stores billions of messages on behalf of millions of end users, while
meeting customer requirements for availability, latency, consistency,
durability, and affordability.

Coinciding with Hibari's development and fine tuning for this GB
webmail service, the system was also deployed as a storage solution
for two major Asian carriers' mobile social networking services. In
this context, Hibari stores user profile data as well as digital goods
of varying types and sizes.

[[hibari-benefits-by-user]]

Hibari Benefits for Developers, System Administrators, and Businesses
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For application developers, Hibari offers a distinctive set of
benefits not often found in distributed key-value stores:

- Strong data consistency guarantees that relieve client applications
  of the burden of managing potential inconsistencies.
- Micro-transaction support that simplifies the creation of powerful
  applications.
- Per-object custom flags that facilitate flexible, service-specific
  application design.
- Support for a variety of API implementations and development
  languages.

For system administrators, Hibari provides valuable operational
automations that simplify data management in a dynamic storage
environment:

- Automatic data replication.
- Automatic failover when a node goes down.
- Automatic repair when a failed node comes back up.
- Automatic rebalancing of data as a cluster grows or shrinks.

For businesses as a whole, Hibari offers affordable Big Data
scalability while delivering the high availability and low latencies
that service users demand. Hibari is an appropriate storage solution
for a range of data-intensive service scenarios including but not
limited to large-scale messaging, social media, and archiving. Hibari
offers particular value in environments that require strong data
consistency and/or high performance across a variety of object types
and sizes.
