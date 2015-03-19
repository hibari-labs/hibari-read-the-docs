Introduction
============

.. caution:: This document is under re-construction -- beware!

The Problem
-----------

There exists a dichotomy in modern storage products. Commodity storage
is inexpensive, but unreliable. Enterprise storage is expensive, but
reliable. Large capacities are present in both enterprise and
commodity class. The problem, then, becomes how to leverage
inexpensive commodity hardware to achieve high capacity enterprise
class reliability at a fraction of the cost.

This problem space has been researched extensively, especially in the
last few years: in academia, the commercial sector, and by open source
community.  Hibari uses techniques and algorithms from this research
to create a solution which is reliable, cost effective, and scalable.

Key-Value Store
---------------

Hibari is key-value store. If a key-value store were represented as
an SQL table, it would be defined as:

[[sql-definition-key-value]]

SQL-like definition of a generic key value store
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: sql

   CREATE TABLE foo (
       BLOB key;
       BLOB value;
   ) PRIMARY KEY key;

In truth, each key stored in Hibari has three additional fields
associated with it. See xref:hibari-data-model[] and
link:hibari-contributor-guide.en.html[Hibari Contributor's Guide] for
details.

[[hibari-origins]]

Hibari's Origins
----------------

Hibari was originally written by Cloudian, Inc., formerly Gemini
Mobile Technologies, to support mobile messaging and email services.
Hibari was released outside of Cloudian under the Apache Public
License version 2.0 in July 2010.

Hibari has been deployed by multiple telecom carriers in Asia and
Europe. Hibari may lack some features such as monitoring, event and
alarm management, and other "production environment" support services.
Since telecom operator has its own data center support infrastructure,
Hibari's development has not included many services that would be
redundant in a carrier environment.

We hope that Hibari's release to the open source community will close
those functional gaps as Hibari spreads outside of carrier data
centers.

Summary of Hibari's Main Features
---------------------------------

- A Hibari cluster is a distributed system.
- A Hibari cluster is linearly scalable.
- A Hibari cluster is highly available.
- All updates are durable.
- All updates are strongly consistent.
- All client operations are lockless.
- A Hibari cluster's performance is excellent.
- Multiple client access protocols are available.
- Data is repaired automatically after a server failure.
- Cluster configuration can be changed at any time.
- Data is automatically rebalanced.
- Heterogeneous hardware support is easy.
- Micro-transactions simplify creation of robust client applications.
- Per-table configurable performance options are available.

[[acid-base-hibari]]

The "ACID vs. BASE" Spectrum and Hibari
---------------------------------------

.. important::
   We strongly believe that "ACID" and "BASE" properties exist on a
   spectrum and are not exclusively one or the other (black-or-white)
   properties.

Most database users and administrators are familiar with the acronym
ACID: Atomic, Consistent, Independent, and Durable. Now, consider an
alternative method of storing and managing data, BASE:

- Basically available
- Soft state
- Eventually consistent

For an
link:http://queue.acm.org/detail.cfm?id=1394128[exploration of ACID and BASE properties (at ACM Queue)], see:

  BASE: An Acid Alternative
  Dan Pritchett
  ACM Queue, volume 6, number 3 (May/June 2008)
  ISSN: 1542-7730
  http://queue.acm.org/detail.cfm?id=1394128

When both strict ACID and strict BASE properties are placed on a
spectrum, they are at the opposite ends. However, a distributed
database system can fit anywhere in the middle of the spectrum.

A Hibari cluster lies near the ACID end of the ACID/BASE spectrum. In
general, Hibari's design will always favors consistency and durability
of updates at the expense of 100% availability in all situations.

[[cap-theorem-and-hibari]]

The CAP Theorem and Hibari
--------------------------

.. warning::
   Eric Brewer's "CAP Theorem", and its proof by Gilbert and Lynch, is
   a tricky thing.  It's nearly impossible to cleanly apply the purity
   of logic to the dirty world of real, industrial computing systems.
   We strongly suggest that the reader consider the CAP properties as
   a spectrum, one of balances and trade-offs. The distributed
   database world is not black and white, and it is important to know
   where the gray areas are.

See the
link:http://en.wikipedia.org/wiki/CAP_theorem[Wikipedia article about the CAP theorem]
for a summary of the theorem, its proof, and related links.

  CAP Theorem (postulated by Eric Brewer, Inktomi, 2000)
  Wikipedia
  http://en.wikipedia.org/wiki/CAP_theorem

Hibari chooses the C and P of CAP. It utilizes chain replication
technique and it always guarantees strong consistency. Hibari also
includes an Erlang/OTP application specifically for detecting network
partitions, so that when a network partition occurs, the brick nodes
in the opposite side of the partition with the active master will be
removed from the chains to keep the strong consistency guarantee.

See xref:admin-server-and-network-partition[] for details.
