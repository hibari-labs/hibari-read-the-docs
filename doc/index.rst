.. Hibari documentation master file, created by
   sphinx-quickstart on Wed Mar 18 20:30:54 2015.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Hibari DB
=========

A Distributed, Consistent, Ordered Key-Value Store
--------------------------------------------------

Hibari is a distributed, ordered key-value store with strong
consistency guarantee. Hibari is written in Erlang and designed for
being:

- **Fast, Read Optimized:** Hibari serves read and write requests in
  short and predictable latency. Hibari has excellent performance
  especially for read and large value operations

- **High Bandwidth:** Batch and lock-less operations help to achieve
  high throughput while ensuring data consistency and durability

- **Big Data:** Can store Peta Bytes of data by automatically
  distributing data across servers. The largest production Hibari
  cluster spans across 100 of servers

- **Reliable:** High fault tolerance by replicating data between
  servers. Data is repaired automatically after a server failure

Hibari is able to deliver scalable high performance that is
competitive with leading open source NOSQL (Not Only SQL) storage
systems, while also providing the data durability and strong
consistency that many systems lack. Hibari's performance relative to
other NOSQL systems is particularly strong for reads and for large
value (> 200KB) operations.

As one example of real-world performance, in a multi-million user
webmail deployment equipped with traditional HDDs (non SSDs), Hibari
is processing about 2,200 transactions per second, with read latencies
averaging between 1 and 20 milliseconds and write latencies averaging
between 20 and 80 milliseconds.

Distinct Features
-----------------

Unlike many other distributed databases, Hibari uses "*chain
replication methodology*" and delivers distinct features.

- **Ordered Key-Values:** Data is distributed across "chains" by key
  prefixes, then keys within a chain are sorted by lexicographic order

- **Always Guarantees Strong Consistency**: This simplifies creation
  of robust client applications

  * **Compare and Swap (CAS):** key timestamping mechanism that
    facilitates "test-and-set" type operations
  * **Micro-Transaction:** multi-key atomic transactions, within
    range limits

- **Custom Metadata**: per-key custom metadata
- **TTL (Time To Live)**: per-key expiration times

Hibari's Origins
----------------

Hibari was originally written by Cloudian, Inc., formerly Gemini
Mobile Technologies, to support mobile messaging and email services.
Hibari was open-sourced under the Apache Public License version 2.0 in
July 2010.

Hibari has been deployed by multiple telecom carriers in Asia and
Europe. Hibari may lack some features such as monitoring, event and
alarm management, and other "production environment" support services.
Since telecom operator has its own data center support infrastructure,
Hibari's development has not included many services that would be
redundant in a carrier environment.

We hope that Hibari's release to the open source community will close
those functional gaps as Hibari spreads outside of carrier data
centers.

.. tip::
   **What does Hibari mean?** The word "Hibari" means skylark in
   Japanese; the Kanji characters stand for **"cloud bird"**.

A Quick Tour
------------

**TODO**

.. toctree::
   :maxdepth: 2

User Documentation
------------------

.. toctree::
   :maxdepth: 1

   == Application Developer's Guide == <app-dev-guide/index.rst>
   app-dev-guide/intro.rst
   app-dev-guide/why-hibari.rst
   app-dev-guide/getting-started.rst
   app-dev-guide/data-model.rst
   app-dev-guide/client-api-overview.rst
   app-dev-guide/client-api-erlang.rst
   app-dev-guide/client-api-ubf.rst
   app-dev-guide/client-api-thrift.rst
   app-dev-guide/build-from-source.rst
   app-dev-guide/contribute.rst

System Administration
---------------------

.. toctree::
   :maxdepth: 1

   == System Admin Guide == <admin-guide/index.rst>
   admin-guide/intro.rst
   admin-guide/main-features.rst
   admin-guide/building-db.rst
   admin-guide/architecture.rst
   admin-guide/admin-server.rst
   admin-guide/system-info.rst
   admin-guide/life-of-brick.rst
   admin-guide/dynamic-cluster-reconfiguration.rst
   admin-guide/partition-detector.rst
   admin-guide/backup-and-dr.rst
   admin-guide/logging.rst
   admin-guide/hardware-software-considerations.rst
   admin-guide/administering-through-api.rst

Hibari Community
----------------

**TODO**

.. toctree::
   :maxdepth: 2

Inside Hibari
-------------

.. toctree::
   :maxdepth: 1

   == Contributor's Guide == <contributors-guide/index.rst>

Misc
----

.. toctree::
   :maxdepth: 1

   Copyright <copyright.rst>
