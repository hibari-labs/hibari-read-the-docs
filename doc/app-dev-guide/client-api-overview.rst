.. highlight:: erlang

Hibari Client API Overview
==========================

As a key-value database, Hibari provides a simple client API with
primitive operations for inserting, retrieving, and deleting data.
Within certain restrictions, the API also supports compound operations
that optionally can be executed as atomic transactions.

Supported Operations
--------------------

Hibari's client API supports the operations listed below.

Data Insertion
^^^^^^^^^^^^^^

.. erl:module:: brick_simple
.. erl:function:: add(Table, Key, Value [,ExpTime] [,Flags] [,Timeout])

   Adds a key-value pair that does not yet exist, along with optional
   flags.

Successful adding of a new key-value pair::

  > brick_simple:add(tab1, <<"foo">>, <<"Hello, world!">>).
  {ok,1271542959131192}

Failed attempt to add a key that already exists::

  > brick_simple:add(tab1, <<"foo">>, <<"Goodbye, world!">>).
  {key_exists,1271542959131192}

.. erl:function:: replace(Table, Key, Value [,ExpTime] [,Flags] [,Timeout])

   Assigns a new value and/or new flags to a key that already exists.

.. erl:function:: set(Table, Key, Value [,ExpTime] [,Flags] [,Timeout])

   Sets a key-value pair and optional flags regardless of whether the
   key yet exists.

.. erl:function:: rename(Table, Key, NewKey [,ExpTime] [,Flags] [,Timeout])

   Renames a key that already exists.

Successful renaming of a key-value pair::

  > brick_simple:rename(tab1, <<"my/foo">>, <<"my/bar">>).
  {ok,1271543165272987}

``rename`` operation fails if ``key`` and ``newkey`` do not share a
common key prefix::

  > brick_simple:rename(tab1, <<"my/foo">>, <<"her/foo">>).
  ...

See **TODO** (Creating New Table - VarPrefix) for more details.

Data Retrieval
^^^^^^^^^^^^^^

- Retrieve a key and optionally its associated value and flags:

  * link:#brick-simple-get[brick_simple:get/4]

- Retrieve multiple lexicographically contiguous keys and optionally
  their associated values and flags:

  * link:#brick-simple-get-many[brick_simple:get_many/5]

Data Deletion
^^^^^^^^^^^^^

- Delete a key-value pair and associated flags:

  * link:#brick-simple-delete[brick_simple:delete/4]

Compound Operations
^^^^^^^^^^^^^^^^^^^

- Execute a specified list of operations, optionally as an atomic
  transaction (micro-transaction):

  * link:#brick-simple-do[brick_simple:do/4]

Fold Operations
^^^^^^^^^^^^^^^

- Implement a fold operation across all keys in a table:

  * link:#brick-simple-fold-table[brick_simple:fold_table/7]

- Implement a fold operation across all keys having a specified
  prefix:

  * link:#brick-simple-fold-key[brick_simple:fold_key_prefix/9]

.. note::
   Fold operations are performed at client side, not server side.

Check and Swap (CAS)
--------------------

If desired, clients can apply a "check and swap" (or "test and set")
logic to data insertion, retrieval, and deletion operations so that
the operation will be executed only if the target key has the exact
timestamp specified in the request.

Micro-Transaction
-----------------

**TODO**
