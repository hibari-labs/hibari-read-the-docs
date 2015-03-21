Introduction
============

Hibari is a production-ready, distributed, key-value, big data
store. In the emerging field of "NOSQL" solutions to today's
mass-scale data storage challenges, Hibari stands out for several
reasons:

- Hibari is the **only open source key-value database to couple Erlang
  engineering with innovative chain replication technology**. Erlang is
  the ideal programming foundation on which to build a robust,
  high-performance distributed storage solution. Chain replication
  delivers high throughput and availability without sacrificing data
  consistency.
- Hibari is the **only open source key-value database built to the
  exacting standards of the carrier-class telecom sector**, and proven
  in multi-million user telecom production environments.
- Hibari delivers a **distinctive feature matrix** that includes:

  * Per-table options for RAM+disk-based or disk-only value storage
  * Support for per-key expiration times and per-key custom meta-data
  * Support for multi-key atomic transactions, within range limits
  * A key timestamping mechanism that facilitates "test-and-set" type
    operations
  * Automatic data rebalancing as the system scales
  * Support for live code upgrades
  * Multiple client API implementations

This introductory chapter will briefly address the recent emergence of
NOSQL solutions to the challenges posed by the "Big Data" era before
turning to describe more fully the distinctive benefits that Hibari
provides to developers, administrators, and users of data-intensive
applications.

Why NOSQL?
----------

The NOSQL *"movement"* is, first off, not an outright rejection of
traditional relational database management systems (RDBMS) but rather
a growing recognition that today's data environment requires a diverse
storage toolset that is *"Not Only SQL (NOSQL)"*. Relational and NOSQL
data storage solutions should be viewed as complements, with each
approach better suited toward different types of applications and
services.

The main driver of NOSQL has been the proliferation of applications
and services that must store and serve terabytes or petabytes of data,
often while striving to guarantee "always-on" availability and low
latencies for end users. Organizations in many market sectors are
grappling with the advent of Big Data, including but not limited to:

- **Web properties** -- coping with the massive data requirements of
  search, e-commerce, social media, and user-generated content.
- **Telecoms** -- managing and analyzing network logs and call data
  records for multi-millions of subscribers.
- **Utilities** -- managing and analyzing the enormous data volume
  associated with smart grids.
- **Financial services** -- storing and mining customer history data in
  order to analyze and model risk.
- **Retail analytics** -- click-stream analysis and micro-targeting.
- **Biotech** -- genome analysis.

Organizations in these and other data-intensive environments have been
challenged to build data storage systems of unprecedented scale. Many
such organizations have found their needs ill-met by traditional data
storage approaches that center around relational database management
systems and specialized high-end hardware. In particular:

- **Scaling up** a single RDBMS instance doesn't achieve nearly the
  scale required, no matter how high-end the systems or how great the
  expenditure.
- **Scaling out** by sharding the system over multiple RDBMS instances
  entails enormous costs and enormous operational complexity, while at
  the same time forfeiting much of the power of the relational model.

Wanting Big Data capacity without crippling cost and complexity, some
innovative organizations have sought a better way to scale. At the
same time, with an ever-expanding array of data usage scenarios, it's
become apparent that not all scenarios require the complex querying
and management functionality associated with an RDBMS. For some
applications and services, SQL-structuring and strict ACID properties
are overkill. Worse, in some environments they're expensive overkill
that can potentially hamstring service offerings in highly competitive
markets that demand flexibility and responsiveness.

In short, recent years have seen a proliferation of services that
require more data, with less structure.

Not surprisingly, some of the leading web enterprises have been at the
forefront of the NOSQL movement. In particular, Google with its
http://labs.google.com/papers/bigtable.html[BigTable paper] in 2006
and Amazon with its
http://s3.amazonaws.com/AllThingsDistributed/sosp/amazon-dynamo-sosp2007.pdf[Dynamopaper]
in 2007 had a profound effect on the NOSQL market. A number of NOSQL
solutions have drawn inspiration from either BigTable or Dynamo or
both, and in the past couple years several solutions have been
released into the open source community.

While NOSQL data storage solutions vary in their particulars, they
have these basic traits in common:

- A simplified data model. Data models vary across specific solutions,
  and sometimes form the basis of a tripartite classification of NOSQL
  systems into 1) key-value data stores (such as Dynamo and Hibari);
  2) column-oriented data stores (such as BigTable); and 3)
  document-oriented data stores (such as CouchDB). All variants,
  however, are simpler and more flexible in data model than the
  traditional RDBMS. That simplification tends to carry over to client
  APIs as well.
- Distribution across multiple nodes based on commodity
  PCs. Affordable Big Data capacity is achieved by scaling out across
  tens, hundreds, or even thousands of commodity PCs. Data
  partitioning schemes coupled with parallel processing of incoming
  requests deliver the needed high performance.

- Replication of data objects across multiple nodes, to ensure high
  availability in the event of component failures.

For much more on the history, merits, and design issues associated
with NOSQL storage solutions, consult with your favorite search
engine.
