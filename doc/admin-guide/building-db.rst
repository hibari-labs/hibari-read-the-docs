Building A Hibari Database
==========================

=== Defining a Schema

Hibari is a key-value database.  Unlike a relational DBMS, Hibari
applications do not need to create a schema.  The only application
requirement is that all its tables be created in advance, see
xref:creating-new-tables[] below.

[[hibari-data-model]]
=== The Hibari Data Model

If a Hibari table were represented within an SQL database, it would
look something like this:

[[sql-definition-hibari]]
.SQL-like definition of a Hibari table
include::texts-src/hibari-sql-definition.txt[]

Hibari table names use the Erlang data type ``atom''.  The types of
all key-related attributes are presented below.

.Types of Hibari table key-value attributes
include::texts-src/hibari-key-value-attrs.txt[]

include::texts-src/hibari-key-value-attrs-expl.txt[]

The practical constraints on maximum value blob size are affected by
total blob size and frequency of large blob access.  For example,
storing an occasional 64MB value blob is different than a 100% write
workload of 100% 64MB value blobs.  The Hibari client API does not
have a method to update or fetch less than the entire value blob, so a
brick can be blocked for many seconds if it tried to operate on (for
example) even a single 4GB blob.  In addition, other processes can be
blocked by `'busy_dist_port'` events while processing big value blobs.

=== Hibari's Client Operations

Hibari's basic client operations are enumerated below.

`add`:: Set a key/value/expiration/flags only if the key does not already exist.
`delete`:: Delete a key
`get`:: Get a key's timestamp and value
`get_many`:: Get a range of keys
`replace`:: Set a key/value/expiration/flags only if the key does exist
`set`:: Set a key/value/expiration/flags
`txn`:: Start of a micro-transaction

Each operation can be accompanied by operation-specific flags.  Some
of these flags include:

`witness`:: Do not return the value blob. (`get`, `get_many`)
`must_exist`:: Abort micro-transaction if key does not exist.
`must_not_exist`:: Abort micro-transaction if key does exist.
`{testset, TS}`:: Perform the action only if the key's current timestamp
exactly matches `TS`. (`delete`, `replace`, `set`, micro-transaction)

For details of these operations and lesser-used per-operation flags,
see:

* xref:micro-transactions[]
* link:hibari-contributor-guide.en.html[Hibari Contributor's Guide]

=== Indexes

Hibari does not support automatic indexing of value blobs.  If an
application requires indexing, the application must build and maintain
those indexes.

[[creating-new-tables]]
=== Creating New Tables

New tables can be created by two different methods:

* Via the Admin Server's status server.  Follow the "Add a table" link
  at the bottom.
* Using the Erlang shell.

For details on the Erlang shell API and detailed explanations of the
table options presented in the Admin server's HTTP interface, see the
link:hibari-contributor-guide.en.html[Hibari Contributor's Guide]
