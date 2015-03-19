The Hibari Data Model
=====================

If a Hibari table were represented within an SQL database, it would
look something like this:

[[sql-definition-hibari]]

include::texts-src/hibari-sql-definition.txt[]

Hibari table names use the Erlang data type ``atom''.  The types of
all key-related attributes are presented below.

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
