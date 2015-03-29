.. highlight:: erlang
.. erl:module:: brick_simple

Client API: Native Erlang
=========================

Data Insertion
--------------

- Add a key-value pair that does not yet exist, along with optional
  flags:

  * link:#brick-simple-add[brick_simple:add/6]

- Assign a new value and/or new flags to a key that already exists:

  * link:#brick-simple-replace[brick_simple:replace/6]

- Rename a key that already exists:

  * link:#brick-simple-rename[brick_simple:rename/6]

- Set a key-value pair and optional flags regardless of whether the
  key yet exists:

   * link:#brick-simple-set[brick_simple:set/6]

Data Retrieval
--------------

- Retrieve a key and optionally its associated value and flags:

  * link:#brick-simple-get[brick_simple:get/4]

- Retrieve multiple lexicographically contiguous keys and optionally
  their associated values and flags:

  * link:#brick-simple-get-many[brick_simple:get_many/5]

Data Deletion
-------------

- Delete a key-value pair and associated flags:

  * link:#brick-simple-delete[brick_simple:delete/4]

Compound Operations
-------------------

- Execute a specified list of operations, optionally as an atomic
  transaction (micro-transaction):

  * link:#brick-simple-do[brick_simple:do/4]

If desired, clients can apply a "test 'n set" logic to data insertion,
retrieval, and deletion operations so that the operation will be
executed only if the target key has the exact timestamp specified in
the request.

Fold Operations
---------------

- Implement a fold operation across all keys in a table:

  * link:#brick-simple-fold-table[brick_simple:fold_table/7]

- Implement a fold operation across all keys having a specified
  prefix:

  * link:#brick-simple-fold-key[brick_simple:fold_key_prefix/9]

.. note::
   Fold operations are performed at client side, not server side.


brick_simple:add/6
------------------

Adds ``Key`` and ``Value`` pair (and optional ``Flags``) to the table
``Table`` if the key does not already exist. The operation will fail
if ``Key`` already exists.

.. erl:function:: add(Table, Key, Value)
.. erl:function:: add(Table, Key, Value, Flags)
.. erl:function:: add(Table, Key, Value, Timeout)
.. erl:function:: add(Table, Key, Value, ExpTime, Flags, Timeout)

   :param Table: Name of the table to which to add the key-value pair

      - ``-type table() :: atom()``

   :type Table: table()

   :param Key: Key to add to the table, in association with a paired value

      - ``-type key() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Key: key()

   .. note::
      While the ``Key`` may be specified as either ``iolist()`` or
      ``binary()``, it will be converted into binary before operation
      execution. The same is true of ``Value``.

   :param Value: Value to associate with the key

      - ``-type val() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Value: val()

   :param ExpTime:

      - Time at which the key will expire, expressed as a Unix
        ``time_t()``.
      - **Optional;** defaults to 0 (no expiration).
      - ``-type exp_time() :: time_t()``
      - ``-type time_t() :: integer()``

   :type ExpTime: exp_time()

   :param Flags:

      - List of operational flags to apply to the ``add`` operation,
        and/or custom property flags to associate with the key-value
        pair in the database. Heavy use of custom property flags is
        discouraged due to RAM-based storage
      - **Optional;** defaults to empty list

      - ``-type flags_list() :: [do_op_flag() | property()]``
      - ``-type do_op_flag() :: 'value_in_ram'``

        * Store the value blob in RAM, overriding the default storage
          location of the brick

          .. note::
             ``'value_in_ram'`` flag have not been extensively tested

      - ``-type property() :: atom() | {term(), term()}``

   :type Flags: flags_list()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success return**

   :rtype: ``{'ok', timestamp()}``

   **Error returns**

   :rtype: ``{'key_exists', timestamp()}``

      - The operation failed because the key already exists.
      - ``-type timestamp() :: integer()``

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_op_flag()`` was
        found in the ``Flags`` argument.

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key is currently length zero and therefore unavailable.

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain.
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful adding of a new key-value pair::

  > brick_simple:add(tab1, <<"foo">>, <<"Hello, world!">>).
  {ok,1271542959131192}

Failed attempt to add a key that already exists::

  > brick_simple:add(tab1, <<"foo">>, <<"Goodbye, world!">>).
  {key_exists,1271542959131192}

Successful adding of a new key-value pair, with value to be stored in
RAM regardless of brick's default storage setting::

  > brick_simple:add(tab1, "foo1", "this is value1", ['value_in_ram']).
  {ok,1271542959131192}

Successful adding of a new key-value pair, using a non-default
operation timeout::

  > brick_simple:add(tab1, "foo2", "this is value2", 20000).
  {ok,1271542959131192}


brick_simple:replace/6
----------------------

Replace ``Key`` and ``Value`` pair (and optional ``Flags``) in the
table ``Table`` if the key already exists. The operation will fail if
``Key`` does not already exist

.. erl:function:: replace(Table, Key, Value)
.. erl:function:: replace(Table, Key, Value, Flags)
.. erl:function:: replace(Table, Key, Value, Timeout)
.. erl:function:: replace(Table, Key, Value, ExpTime, Flags, Timeout)

   :param Table: Name of the table in which to replace the key-value pair.

      - ``-type table() :: atom()``

   :type Table: table()

   :param Key:
      Key to replace in the table, in association with a new paired
      value

      - ``-type key() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   .. note::
      While the ``Key`` may be specified as either ``iolist()`` or
      ``binary()``, it will be converted into binary before operation
      execution. The same is true of ``Value``.

   :param Value: Value to associate with the key

      - ``-type val() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Value: val()

   :param ExpTime:

      - Time at which the key will expire, expressed as a Unix
        ``time_t()``.
      - **Optional;** defaults to 0 (no expiration).
      - ``-type exp_time() :: time_t()``
      - ``-type time_t() :: integer()``

   :type ExpTime: exp_time()

   :param Flags:

      - List of operational flags to apply to the ``replace``
        operation, and/or custom property flags to associate with the
        key-value pair in the database. Heavy use of custom property
        flags is discouraged due to RAM-based storage
      - **Optional;** defaults to empty list

      - ``-type flags_list() :: [do_op_flag() | property()]``
      - ``-type do_op_flag() :: {'testset', timestamp()} | 'value_in_ram'``
        ``{'exp_time_directive', 'keep' | 'replace'} |``
        ``{'attrib_directive', 'keep' | 'replace'}``
      - ``-type timestamp() = integer()``
      - ``-type property() :: atom() | {term(), term()}``
      - Operational flag usage

        * ``{'testset', timestamp()}``

          * Fail the operation if the existing key's timestamp is not
            exactly equal to ``timestamp()``.  If used inside a
            link:#brick-simple-do[micro-transaction], abort the
            transaction if the key's timestamp is not exactly equal to
            ``timestamp()``

        * ``{'exp_time_directive', 'keep' | 'replace'}``

          * Default to ``'replace'``
          * Specifies whether the ``ExpTime`` is kept from the old key
            value pair or replaced with the ``ExpTime`` provided in
            the replace operation

        * ``{'attrib_directive', 'keep' | 'replace'}``

          * Default to ``'replace'``
          * Specifies whether the custom properties are kept from the
            old key value pair or replaced with the custom properties
            provided in the replace operation
          * If kept, the custom properties remain unchanged. If you
            specify custom properties explicitly in the replace
            operation, Hibari adds them to the resulting key value
            pair
          * If replaced, all original custom properties are deleted,
            and then Hibari adds the custom properties in the replace
            operation to the resulting key value pair

        * ``'value_in_ram'``

          * Store the value blob in RAM, overriding the default
            storage location of the brick

          .. note::
             ``'value_in_ram'`` flag have not been extensively tested

   :type Flags: flags_list()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success return**

   :rtype: ``{'ok', timestamp()}``

   **Error returns**

   :rtype: ``'key_not_exists'``

      - The operation failed because the key does not exist
      - ``-type timestamp() :: integer()``

   :rtype: ``{'ts_error', timestamp()}``

      - The operation failed because the ``{'testset', timestamp()}``
        flag was used and there was a timestamp mismatch. The
        ``timestamp()`` in the return is the current value of the
        existing key's timestamp.
      - ``timestamp() = integer()``

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_op_flag()`` was
        found in the ``Flags`` argument.

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key is currently length zero and therefore unavailable.

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain.
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful replacement of a key-value pair::

  > brick_simple:replace(tab1, <<"foo">>, <<"Goodbye, world!">>).
  {ok,1271543165272987}

Failed attempt to replace a key that does not yet exist::

  > brick_simple:replace(tab1, <<"key3">>, <<"new and improved value">>).
  key_not_exist

Successful replacement of a key-value pair, with value to be stored in
RAM regardless of brick's default storage setting::

  > brick_simple:replace(tab1, "foo", "You again, world!", ['value_in_ram']).
  {ok,1271543165272987}

Failed attempt to replace a key for which we have incorrectly
specified its current timestamp::

  > brick_simple:replace(tab1, "foo", "Whole new value", [{'testset', 12345}]).
  {ts_error,1271543165272987}

Successful replacement of a key-value pair for which we have correctly
specified its current timestamp::

  > brick_simple:replace(tab1, "foo", "Whole new value", [{'testset', 1271543165272987}]).
  {ok,1271543165272988}

Successful replacement of a key-value pair, using a non-default
operation timeout::

  > brick_simple:replace(tab1, "foo", "Foo again?", 30000).
  {ok,1271543165272989}

brick_simple:set/6
------------------

Set ``Key`` and ``Value`` pair (and optional ``Flags``) in the table
``Table``, regardless of whether or not the key already exists.

.. erl:function:: set(Table, Key, Value)
.. erl:function:: set(Table, Key, Value, Flags)
.. erl:function:: set(Table, Key, Value, Timeout)
.. erl:function:: set(Table, Key, Value, ExpTime, Flags, Timeout)

   :param Table: Name of the table to which to set the key-value pair

      - ``-type table() :: atom()``

   :type Table: table()

   :param Key:
      Key to set in to the table, in association with a paired value

      - ``-type key() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Key: key()

   .. note::
      While the ``Key`` may be specified as either ``iolist()`` or
      ``binary()``, it will be converted into binary before operation
      execution. The same is true of ``Value``.

   :param Value: Value to associate with the key

      - ``-type val() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :param ExpTime:

      - Time at which the key will expire, expressed as a Unix
        ``time_t()``.
      - **Optional;** defaults to 0 (no expiration).
      - ``-type exp_time() :: time_t()``
      - ``-type time_t() :: integer()``

   :type ExpTime: exp_time()

   :param Flags:

      - List of operational flags to apply to the ``set`` operation,
        and/or custom property flags to associate with the key-value
        pair in the database. Heavy use of custom property flags is
        discouraged due to RAM-based storage
      - **Optional;** defaults to empty list

      - ``-type flags_list() :: [do_op_flag() | property()]``
      - ``-type do_op_flag() :: {'testset', timestamp()} | 'value_in_ram'``
        ``| {'exp_time_directive', 'keep' | 'replace'}``
        ``| {'attrib_directive', 'keep' | 'replace'}``
      - ``-type timestamp() :: integer()``
      - ``-type property() :: atom() | {term(), term()}``
      - Operational flag usage

        * ``{'testset', timestamp()}``

          * Fail the operation if the existing key's timestamp is not
            exactly equal to ``timestamp()``.  If used inside a
            link:#brick-simple-do[micro-transaction], abort the
            transaction if the key's timestamp is not exactly equal to
            ``timestamp()``. Using this flag with ``set`` will result
            in an error if the key does not already exist or if the
            key exists but has a non-matching timestamp.

        * ``{'exp_time_directive', 'keep' | 'replace'}``

          * Default to ``'replace'``
          * Specifies whether the ``ExpTime`` is kept from the old key
            value pair or replaced with the ``ExpTime`` provided in
            the replace operation

        * ``{'attrib_directive', 'keep' | 'replace'}``

          * Default to ``'replace'``
          * Specifies whether the custom properties are kept from the
            old key value pair or replaced with the custom properties
            provided in the set operation
          * If kept, the custom properties remain unchanged. If you
            specify custom properties explicitly in the set
            operation, Hibari adds them to the resulting key value
            pair
          * If replaced, all original custom properties are deleted,
            and then Hibari adds the custom properties in the set
            operation to the resulting key value pair

        * ``'value_in_ram'``

          * Store the value blob in RAM, overriding the default
            storage location of the brick

          .. note::
             ``'value_in_ram'`` flag have not been extensively tested

   :type Flags: flags_list()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success return**

   :rtype: ``{'ok', timestamp()}``

   **Error returns**

   :rtype: ``'key_not_exists'``

      - The operation failed because the ``{'testset', timestamp()}``
        flag was used and  key does not exist
      - ``-type timestamp() :: integer()``

   :rtype: ``{'ts_error', timestamp()}``

      - The operation failed because the ``{'testset', timestamp()}``
        flag was used and there was a timestamp mismatch. The
        ``timestamp()`` in the return is the current value of the
        existing key's timestamp.
      - ``timestamp() = integer()``

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_op_flag()`` was
        found in the ``Flags`` argument.

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key is currently length zero and therefore unavailable.

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain.
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful setting of a key-value pair::

  > brick_simple:set(tab1, <<"key4">>, <<"cool value">>).
  {ok,1271542959131192}

Successful setting of a key-value pair, with value to be stored in RAM
regardless of brick's default storage setting::

  > brick_simple:set(tab1, "goo", "value6", ['value_in_ram']).
  {ok,1271542959131193}

Failed attempt to set a key-value pair, when we have used the
``testset`` flag but the key does not yet exist::

  > brick_simple:set(tab1, "boo", "hoo", [{'testset', 1271543165272987}]).
  key_not_exist

Successful setting of a key-value pair, when we have used the
``testset`` flag and the key does already exist and its timestamp
matches our specified timestamp::

  > brick_simple:set(tab1, "goo", "value7", [{'testset', 1271543165272432}]).
  {ok,1271543165272433}

brick_simple:rename/6
---------------------

Rename ``Key``, ``Value`` pair, and ``Flags`` to ``NewKey`` in the
table ``Table`` if the key already exists. The operation will fail if:

- ``Key`` does not already exist
- ... or ``Key`` and ``NewKey`` do not share a common key prefix.
  (See **TODO** (Creating New Table - VarPrefix) for more details)

.. erl:function:: rename(Table, Key, NewKey)
.. erl:function:: rename(Table, Key, NewKey, Flags)
.. erl:function:: rename(Table, Key, NewKey, Timeout)
.. erl:function:: rename(Table, Key, NewKey, ExpTime, Flags, Timeout)

   :param Table:
      Name of the table to which to rename the key-value pair

      - ``-type table() :: atom()``

   :type Table: table()

   :param Key:
      Key to rename in to the table, in association with a paired value

      - ``-type key() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Key: key()

   .. note::
      While the ``Key`` may be specified as either ``iolist()`` or
      ``binary()``, it will be converted into binary before operation
      execution. The same is true of ``NewKey``

   :param NewKey:
      NewKey in the table, in association with an existing paired
      value

      - ``-type val() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :param ExpTime:

      - Time at which the key will expire, expressed as a Unix
        ``time_t()``.
      - **Optional;** defaults to 0 (no expiration).
      - ``-type exp_time() :: time_t()``
      - ``-type time_t() :: integer()``

   :type ExpTime: exp_time()

   :param Flags:

      - List of operational flags to apply to the ``rename``
        operation, and/or custom property flags to associate with the
        key-value pair in the database. Heavy use of custom property
        flags is discouraged due to RAM-based storage
      - **Optional;** defaults to empty list

      - ``-type flags_list() :: [do_op_flag() | property()]``
      - ``-type do_op_flag() :: {'testset', timestamp()} | 'value_in_ram'``
        ``| {'exp_time_directive', 'keep' | 'replace'}``
        ``| {'attrib_directive', 'keep' | 'replace'}``
      - ``-type timestamp() :: integer()``
      - ``-type property() :: atom() | {term(), term()}``
      - Operational flag usage

        * ``{'testset', timestamp()}``

          * Fail the operation if the existing key's timestamp is not
            exactly equal to ``timestamp()``.  If used inside a
            link:#brick-simple-do[micro-transaction], abort the
            transaction if the key's timestamp is not exactly equal to
            ``timestamp()``.

        * ``{'exp_time_directive', 'keep' | 'replace'}``

          * Default to ``'keep'``
          * Specifies whether the ``ExpTime`` is kept from the old key
            value pair or replaced with the ``ExpTime`` provided in
            the rename operation

        * ``{'attrib_directive', 'keep' | 'replace'}``

          * Default to ``'keep'``
          * Specifies whether the custom properties are kept from the
            old key value pair or replaced with the custom properties
            provided in the rename operation
          * If kept, the custom properties remain unchanged. If you
            specify custom properties explicitly in the rename
            operation, Hibari adds them to the resulting key value
            pair
          * If replaced, all original custom properties are deleted,
            and then Hibari adds the custom properties in the rename
            operation to the resulting key value pair

        * ``'value_in_ram'``

          * Store the value blob in RAM, overriding the default
            storage location of the brick

          .. note::
             ``'value_in_ram'`` flag have not been extensively tested

   :type Flags: flags_list()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()


   **Success return**

   :rtype: ``{'ok', timestamp()}``

   **Error returns**

   :rtype: ``'key_not_exists'``

      - The operation failed because the key does not exist or because
        key and the new key are equal
      - ``-type timestamp() :: integer()``

   :rtype: ``{'ts_error', timestamp()}``

      - The operation failed because the ``{'testset', timestamp()}``
        flag was used and there was a timestamp mismatch. The
        ``timestamp()`` in the return is the current value of the
        existing key's timestamp.
      - ``timestamp() = integer()``

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_op_flag()`` was
        found in the ``Flags`` argument.

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key and the new key is currently length zero and
        therefore unavailable.

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain.
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful renaming of a key-value pair::

  > brick_simple:rename(tab1, <<"foo">>, <<"bar">>).
  {ok,1271543165272987}

Failed attempt to rename a key that does not yet exist::

  > brick_simple:rename(tab1, <<"key3">>, <<"bar">>).
  key_not_exist

Successful renaming of a key-value pair, with value to be stored in
RAM regardless of brick's default storage setting::

  > brick_simple:rename(tab1, "foo", "bar", ['value_in_ram']).
  {ok,1271543165272987}

Failed attempt to rename a key for which we have incorrectly
specified its current timestamp::

  > brick_simple:rename(tab1, "foo", "bar", [{'testset', 12345}]).
  {ts_error,1271543165272987}

Successful renaming of a key-value pair for which we have correctly
specified its current timestamp::

  > brick_simple:rename(tab1, "foo", "bar", [{'testset', 1271543165272987}]).
  {ok,1271543165272988}

Successful renaming of a key-value pair, using a non-default
operation timeout::

  > brick_simple:rename(tab1, "foo", "bar", 30000).
  {ok,1271543165272989}

brick_simple:get/4
------------------

From table ``Table``, retrieve ``Key`` and specified attributes of the
key (as determined by ``Flags``).

.. erl:function:: get(Table, Key)
.. erl:function:: get(Table, Key, Flags)
.. erl:function:: get(Table, Key, Timeout)
.. erl:function:: get(Table, Key, Flags, Timeout)

   :param Table:
      Name of the table from which to retrieve the key-value pair

      - ``-type table() :: atom()``

   :type Table: table()

   :param Key:
      Key to retrieve from to the table

      - ``-type key() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Key: key()

   .. note::
      While the ``Key`` may be specified as either ``iolist()`` or
      ``binary()``, it will be converted into binary before operation
      execution

   :param Flags:

      - List of operational flags to apply to the ``get`` operation.
      - **Optional;** defaults to empty list

      - ``-type flags_list() :: [do_op_flag()]``
      - ``-type do_op_flag() :: 'get_all_attribs' | 'witness'``
        ``| {'testset', timestamp()}``
        ``| 'must_exist' | 'must_not_exist'``
      - ``-type timestamp() :: integer()``
      - Operational flag usage


        * ``'get_all_attribs'``

          * Return all attributes of the key. May be used in
            combination with the ``witness`` flag

        * ``'witness'``

          * Do not return the value blob in the result. This flag will
            guarantee that the brick does not require disk access to
            satisfy this request

        * ``{'testset', timestamp()}``

          * Fail the operation if the key's timestamp is not exactly
            equal to ``timestamp()``. If used inside a
            link:#brick-simple-do[micro-transaction], abort the
            transaction if the key's timestamp is not exactly equal to
            ``timestamp()``.
          * This flag has priority over the ``'must_exist'`` and
            ``'must_not_exist'`` flags

        * ``'must_exist'``

          * For use inside a link:#brick-simple-do[micro-transaction]:
            abort the transaction if the key does not exist

        * ``'must_not_exist'``

          * For use inside a link:#brick-simple-do[micro-transaction]:
            abort the transaction if the key exists. This flag may be
            useful when the relationship between two or more keys is
            important to the client application

   :type Flags: flags_list()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success returns**

   :rtype: ``{'ok', timestamp(), val()}``

      - Success return when the get request uses neither the
        ``'witness'`` flag nor the ``'get_all_attribs'`` flag
      - ``-type timestamp() :: integer()``
      - ``-type val() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist()  :: [char() | binary() | iolist()]``

   :rtype: ``{'ok', timestamp()}``

      - Success return when the get uses ``'witness'`` but not
        ``'get_all_attribs'``

   :rtype: ``{'ok', timestamp(), exp_time(), proplist()}``

      - Success return when the get uses both ``'witness'`` and
        ``'get_all_attribs'``
      - ``-type exp_time() :: time_t()``
      - ``-type proplist() :: [property()]``
      - ``-type property() :: atom() | {term(), term()}``

   :rtype: ``{'ok', timestamp(), val(), exp_time(), proplist()}``

      - Success return when the get uses ``'get_all_attribs'`` but not
        ``'witness'``
      - ``-type exp_time() :: time_t()``

   .. note::
      When a ``proplist()`` is returned, one of the properties in the
      list will always be ``{val_len, Size::integer()}``, where
      ``Size`` is the size of the value blob in bytes

   **Error returns**

   :rtype: ``'key_not_exist'``

      - The operation failed because the key does not exist.

   :rtype: ``{'ts_error', timestamp()}``

      - The operation failed because the ``{'testset', timestamp()}``
        flag was used and there was a timestamp mismatch. The
        ``timestamp()`` in the return is the current value of the
        existing key's timestamp.

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_op_flag()`` was
        found in the ``Flags`` argument

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key is currently length zero and therefore unavailable.

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain.
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful retrieval of a key-value pair::

  > brick_simple:get(tab1, "goo").
  {ok,1271543165272432,<<"value7">>}

Successful retrieval of a key without its associated value blob::

  > brick_simple:get(tab1, "goo", ['witness']).
  {ok,1271543165272432}

Failed attempt to retrieve a key that does not exist::

  > brick_simple:get(tab1, "moo").
  key_not_exist

brick_simple:get_many/5
-----------------------

Get many keys from a single chain in the table ``Table``, up to a
maximum of ``MaxNum`` keys. Keys are returned in lexicographic sorting
order starting with the first key _after_ the key specified by the
``Key`` argument. The return list includes a boolean value indicating
whether or not there are more keys after the last key of the return
results.

.. important::
   A single ``get_many()`` function call cannot be used to retrieve
   keys from across multiple storage chains. The consistent hash of
   ``Key`` will send the ``get_many`` operation to the tail brick in a
   single chain; all keys returned will come from that single brick
   only.

.. erl:function:: get_many(Table, Key, MaxNum)
.. erl:function:: get_many(Table, Key, MaxNum, Flags)
.. erl:function:: get_many(Table, Key, MaxNum, Timeout)
.. erl:function:: get_many(Table, Key, MaxNum, Flags, Timeout)

   :param Table:
      Name of the table to which to retrieve the key-value pair

      - ``-type table() :: atom()``

   :type Table: table()

   :param Key:
      Key after which to start the ``get_many`` retrieval, proceeding
      in lexicographic order with the first key after the specified
      ``Key``

      - ``-type key() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Key: key()

   .. note::
      While the ``Key`` may be specified as either ``iolist()`` or
      ``binary()``, it will be converted into binary before operation
      execution

   :param MaxNum: Maximum number of keys to return
   :type  MaxNum: integer()

   :param Flags:

      - List of operational flags to apply to the ``get_many``
        operation.
      - **Optional;** defaults to empty list

      - ``-type flags_list() :: [do_op_flag()]``
      - ``-type do_op_flag() :: 'get_all_attribs' | 'witness'``
        ``| {'binary_prefix', binary()}``
        ``| {'max_bytes', integer()}``
        ``| {'max_num', integer()}``
      - ``-type timestamp() :: integer()``
      - ``-type property() :: atom() | {term(), term()}``
      - Operational flag usage

   :type Flags: flags_list()

        * ``'get_all_attribs'``

          * Return all attributes of the key. May be used in
            combination with the ``witness`` flag

        * ``'witness'``

          * Do not return the value blob in the result. This flag will
            guarantee that the brick does not require disk access to
            satisfy this request

        * ``{'binary_prefix', binary()}``

          * Return only keys that have a binary prefix that is exactly
            equal to ``binary()``

        * ``{'max_bytes', integer()}``

          * Return only as many keys as the sum of the sizes of their
            corresponding value blobs does not exceed ``integer()``
            bytes. If this flag is not explicity specified in a client
            request, the value defaults to 2GB

        * ``{'max_num', integer()}``

          * Maxinum number of keys to return. Defaults to 10. Note:
            This flag is duplicative of the MaxNum argument in
            purpose

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success returns**

   :rtype: ``{ok, {[{key(), timestamp(), val()}], boolean()}}``

      - Success return when the ``get_many`` request uses neither the
        ``'witness'`` flag nor the ``'get_all_attribs'`` flag
      - ``-type timestamp() :: integer()``
      - ``-type val() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``iolist() :: [char() | binary() | iolist()]``

   :rtype: ``{ok, {[{key(), timestamp()}], boolean()}}``

      - Success return when the ``get_many`` uses ``'witness'`` but
        not ``'get_all_attribs'``

   :rtype: ``{ok, {[{key(), timestamp(), exp_time(), proplist()}], boolean()}}``

      - Success return when the ``get_many`` uses both ``'witness'``
        and ``'get_all_attribs'``
      - ``-type exp_time() :: time_t()``
      - ``-type proplist() :: [property()]``
      - ``property() :: atom() | {term(), term()}``

   :trype: ``{ok, {[{key(), timestamp(), val(), exp_time(), proplist()}], boolean()}}``

      - Success return when the ``get_many`` uses
        ``'get_all_attribs'`` but not ``'witness'``
      - ``exp_time() :: time_t()``

   .. note::
      The boolean at the end of the success return indicates whether
      or not the chain has more keys lexicographically after the last
      key in the return (``true`` for yes, ``false`` for no). When a
      ``proplist()`` is returned, one of the properties in the list
      will always be ``{val_len, Size::integer()}``, where ``Size`` is
      the size of the value blob in bytes.

   **Error returns**

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_op_flag()`` was
        found in the ``Flags`` argument.

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key is currently length zero and therefore unavailable.

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain.
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful retrieval of all keys from a table that currently has only
two keys. The boolean `false' indicates that there are no keys
following the ``foo`` key::

  > brick_simple:get_many(tab1, "", 5).
  {ok,{[{<<"another">>,1271543102911775,<<"yes!">>},
        {<<"foo">>,1271543165272987,<<"Foo again?">>}],
       false}}

Successful retrieval of all keys from a table that currently has only
two keys, using the ``witness`` flag in the request::

  > brick_simple:get_many(tab1, "", 5, ['witness']).
  {ok,{[{<<"another">>,1271543102911775},
        {<<"foo">>,1271543165272987}],
       false}}

Successful retrieval of all keys from a table that currently has only
two keys, using the ``get_all_attribs`` flag in the request.::

  > brick_simple:get_many(tab1, "", 5).
  {ok,{[{<<"another">>,1271543102911775,<<"yes!">>,0,[{val_len,4}]},
        {<<"foo">>,1271543165272987,<<"Foo again?">>,0,[{val_len,6}]}],
       false}}

brick_simple:delete/4
---------------------

Delete key `Key` from the table `Table`. The operation will fail if
``Key`` does not already exist

.. erl:function:: delete(Table, Key)
.. erl:function:: delete(Table, Key, Flags)
.. erl:function:: delete(Table, Key, Timeout)
.. erl:function:: delete(Table, Key, Flags, Timeout)

   :param Table:
      Name of the table from which to delete the key-value pair

      - ``-type table() :: atom()``

   :type Table: table()

   :param Key:
      Key to delete from the table

      - ``-type key() :: iodata()``
      - ``-type iodata() :: iolist() | binary()``
      - ``-type iolist() :: [char() | binary() | iolist()]``

   :type Key: key()

   .. note::
      While the ``Key`` may be specified as either ``iolist()`` or
      ``binary()``, it will be converted into binary before operation
      execution

   :param Flags:

      - List of operational flags to apply to the ``delete``
        operation.
      - **Optional;** defaults to empty list

      - ``-type flags_list() :: [do_op_flag()]``
      - ``-type do_op_flag() :: {'testset', timestamp()}``
        ``| 'must_exist' | 'must_not_exist'``
      - ``-type timestamp() :: integer()``
      - Operational flag usage

        * ``{'testset', timestamp()}``

          * Fail the operation if the existing key's timestamp is not
            exactly equal to ``timestamp()``.  If used inside a
            link:#brick-simple-do[micro-transaction], abort the
            transaction if the key's timestamp is not exactly equal to
            ``timestamp()``. This flag has priority over the
            ``'must_exist'`` and ``'must_not_exist'`` flags

        * ``'must_exist'``

          * For use inside a link:#brick-simple-do[micro-transaction]:
            abort the transaction if the key does not exist

        * ``'must_not_exist'``

          * For use inside a link:#brick-simple-do[micro-transaction]:
            abort the transaction if the key exists. This flag may be
            useful when the relationship between two or more keys is
            important to the client application

   :type Flags: flags_list()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success return**

   :rtype: ``'ok'``

   **Error returns**

   :rtype: ``'key_not_exist'``

      - The operation failed because the key does not exist

   :rtype: ``{'ts_error', timestamp()}``

      - The operation failed because the ``{'testset', timestamp()}``
        flag was used and there was a timestamp mismatch. The
        ``timestamp()`` in the return is the current value of the
        existing key's timestamp.
      - ``timestamp() = integer()``

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_op_flag()`` was
        found in the ``Flags`` argument.

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key is currently length zero and therefore unavailable.

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain.
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful deletion of a key and its associated value and attributes::

  > brick_simple:delete(tab1, <<"foo">>).
  ok

Failed attempt to delete a key that does not exist::

  > brick_simple:delete(tab1, "key6").
  key_not_exist

Failed attempt to delete a key for which we have incorrectly specified
its current timestamp::

  > brick_simple:delete(tab1, "goo", [{'testset', 12345}]).
  {ts_error,1271543165272987}

Successful deletion of a key for which we have correctly specified its
current timestamp::

  > brick_simple:delete(tab1, "goo", [{'testset', 1271543165272987}]).
  ok

Successful deletion of a key, using a non-default operation timeout::

  > brick_simple:delete(tab1, "key3", 30000).
  ok

brick_simple:do/4
-----------------

Send a list of primitive operations to the table ``Table``. They will
be executed at the same time by a Hibari brick. If the first item in
the ``OpList`` is ``brick_server:make_txn()`` then the list of
operations is executed in the context of a micro-transaction: either
all operations will be executed successfully or none will be executed.

We term these "micro"-transactions because they are subject to certain
limitations that apply to all operations that use the
``brick_simple:do()`` API:

- All impacted keys must be in the same table.
- All impacted keys must be in the same chain.
- All operations in the transaction must be sent in a single
  ``brick_simple:do()`` call. Unlike some other databases, it is not
  possible to request a transaction handle and to add operations to
  that transaction in an one-by-one, "ad hoc" manner.

For further information about micro-transactions, see
link:hibari-sysadmin-guide.en.html#micro-transactions[Hibari System
Administrator's Guide, "Micro-Transactions" section].

.. erl:function:: do(Table, OpList)
.. erl:function:: do(Table, OpList, Timeout)
.. erl:function:: do(Table, OpList, OpFlags, Timeout)

   :param Table:
      Name of the table in which to perform the operations

      - ``-type table() :: atom()``

   :type Table: table()

   :param OpList:

      - List of primitive operations to perform. Each primitive is
        invoked using the ``brick_server:make_*()`` API
      - ``-type do_op_list() :: [do1_op()]``
      - ``-type do1_op() ::``

        * ``brick_server:make_add(Key, Value, ExpTime, Flags)``
        * ``brick_server:make_replace(Key, Value, ExpTime, Flags)``
        * ``brick_server:make_set(Key, Value, ExpTime, Flags)``
        * ``brick_server:make_rename(Key, NewKey, ExpTime, Flags)``
        * ``brick_server:make_get(Key, Flags)``
        * ``brick_server:make_get_many(Key, Flags)``
        * ``brick_server:make_delete(Key, Flags)``
        * ``brick_server:make_txn()``

          * Include ``brick_server:make_txn()`` as the first item in
            your ``OpList`` if you want the ``do`` operation to be
            executed as an atomic transaction
          * Note that the arguments for each primitive are the same as
            those for the primitives when they are executed on their
            own, with the exclusion of the ``Tab`` and ``Timeout``
            arguments, both of which serve as arguments to the overall
            ``do`` operation rather than as arguments to the
            primitives. For example, an ``add`` on its own is
            ``brick_simple:add(Tab, Key, Value, ExpTime, Flags,
            Timeout)``, whereas in the context of a ``do`` operation
            an ``add`` primitive is ``brick_server:make_add(Key,
            Value, ExpTime, Flags)``
          * For further information about each primitive, see
            link:#brick-simple-add[brick_simple:add/6],
            link:#brick-simple-replace[brick_simple:replace/6],
            link:#brick-simple-set[brick_simple:set/6],
            link:#brick-simple-rename[brick_simple:rename/6],
            link:#brick-simple-get[brick_simple:get/4],
            link:#brick-simple-get-many[brick_simple:get_many/5], and
            link:#brick-simple-delete[brick_simple:delete/4]

   :type OpList: do_op_list()

   :param OpFlags:

      - List of operational flags to apply to the overall ``do``
        operation.
      - **Optional;** defaults to empty list

      - ``-type do_flags_list() :: [do_flag()]``
      - ``-type do_flag() :: 'fail_if_wrong_role' | 'ignore_role'``
      - Operational flag usage

        * ``'fail_if_wrong_role'``

          * If the 'do' operation is sent to the wrong brick in the
            target  chain (e.g. a 'read' request mistakenly sent to
            the 'head' brick or a 'write' request mistakenly sent to
            the 'tail' brick), fail the transaction immediately. If
            this flag is not used, the default behavior is for the
            incorrect brick to forward the request to the correct
            brick

        * ``'ignore_role'``

          * If this flag is used, then whichever brick receives the
            request  will reply to the request directly, regardless of
            the brick's assigned role

   :type OpFlags: do_flags_list()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success return**

   :rtype: ``[do1_res_ok]``

      - List of ``do1_res_ok``, one for each primitive operation
        specified in the ``do`` request. Return list order corresponds
        to the order in which primitive operations are listed in the
        request's ``OpList``. Note that if the ``do`` request does not
        use transaction semantics, then some individual primitive
        operations may fail without the overall ``do`` operation
        failing
      - Within the return list, possible ``do1_res_ok`` returns to
        each individual primitive operation are the same as the
        possible returns that the primitive operation type could
        generate if it were executed on its own. For example, within
        the ``do`` operation's success return list, the possible
        returns for a primitive ``add`` operation are the same as the
        returns described in the
        link:#brick-simple-add[brick_simple:add/6] section; potential
        returns to a primitive ``replace`` operation are the same as
        those described in the
        link:#brick-simple-replace[brick_simple:replace/6] section; and
        likewise for link:#brick-simple-set[set],
        likewise for link:#brick-simple-rename[rename],
        link:#brick-simple-get[get],
        link:#brick-simple-get-many[get_many], and
        link:#brick-simple-delete[delete].

   **Error returns**

   :rtype: ``{txn_fail, [{integer(), do1_res_fail()}]}``

      - Operation failed because transaction semantics were used in
        the ``do`` request and one or more primitive operations within
        the transaction failed. The ``integer()`` identifies the
        failed primitive operation by its position within the
        request's ``OpList``. For example, a 2 indicates that the
        second primitive listed in the request's ``OpList``
        failed. Note that this position identifier does not count the
        ``txn()`` specifier at the start of  the ``OpList``.
      - ``do1_res_fail()`` indicates the type of failure for the
        failed primitive operation. Possibilities are:

        * ``{'key_exists', timestamp()}``

          * ``-type timestamp() :: integer()``

        * ``'key_not_exist'``
        * ``{'ts_error', timestamp()}``
        * ``'invalid_flag_present'``

   :rtype: ``'invalid_flag_present'``

      - The operation failed because an invalid ``do_flag()`` was
        found in the ``do`` request's ``OpFlags`` argument. Note this
        is a different error than an invalid flag being found within
        an individual primitive

   :rtype: ``'brick_not_available'``

      - The operation failed because the chain that is responsible for
        this key is currently length zero and therefore unavailable

   :rtype: ``{{'nodedown',node()},{'gen_server','call',term()}}``

      - The operation failed because the server brick handling the
        request has crashed or else a network partition has occurred
        between the client and server. The client should resend the
        query after a short delay, on the assumption that the Admin
        Server will have detected the failure and taken steps to
        repair the chain
      - ``-type node() :: atom()``

Examples
^^^^^^^^

Successful ``do`` operation adding two new keys to table ``tab1``,
without transaction semantics::

  > brick_simple:do(tab1, [brick_server:make_add("foo3", "bar3"),
                           brick_server:make_add("foo4", "bar4")]).
  [ok,ok]

Successful creation of two ``get`` primitives ``Do1` and ``Do2`, and
their subsequent combination into a ``do`` request, without
transaction semantics::

  > Do1 = brick_server:make_get("foo").
  {get,<<"foo">>,[]}
  > Do2 = brick_server:make_get("foo2").
  {get,<<"foo2">>,[]}
  > brick_simple:do(tab1, [Do1, Do2]).
  [{ok,1271543102911775,<<"Foo again?">>},key_not_exist]

Failed operation with transaction semantics. Because transaction
semantics are used, the failure of the primitive ``Do2b`` causes the
entire operation to fail::

  > Do1b = brick_server:make_get("foo").
  {get,<<"foo">>,[]}
  > Do2b = brick_server:make_get("foo2", [must_exist]).
  {get,<<"foo2">>,[must_exist]}
  > brick_simple:do(tab1, [brick_server:make_txn(), Do1b, Do2b]).
  {txn_fail,[{2,key_not_exist}]}

brick_simple:fold_table/7
-------------------------

Attempt a fold operation across all keys in a table. For general
information about the Erlang fold function that underlies this
operations, see http://www.erlang.org/doc/man/lists.html#foldl-3.

.. important::
   Do not execute this operation while a data migration is being
   performed

.. erl:function:: fold_table(Table, Fun, Acc, NumItems, Flags)
.. erl:function:: fold_table(Table, Fun, Acc, NumItems, Flags, MaxParallel)
.. erl:function:: fold_table(Table, Fun, Acc, NumItems, Flags, MaxParallel, Timeout)

   :param Table:
      Name of the table across which to perform the fold operation

      - ``-type table() :: atom()``

   :type Table: table()

   :param Fun:
      Function to apply to successive elements of the list

      - ``-type fun_arity_2() :: fun(({ChainName, TupleFromGetMany}, Acc) -> Acc)``

        * ``TupleFromGetMany`` is a single result tuple from a
          link:#brick-simple-get-many[brick_simple:get_many()]
          result. Its format can vary according to the ``Flags``
          argument, which is passed as-is to a ``get_many()`` call. For
          example, if ``Flags`` = ``[]``, then ``TupleFromGetMany``
          will match ``{Key, TS, Value}``. If ``Flags`` = ``[witness]``,
          then ``TupleFromGetMany`` will match ``{Key, TS}``

      - ``Acc``

        * The accumulator term

   :type Fun: fun_arity_2()

   :param Acc: Initial value of the accumulator term
   :type Acc:  term()

   :param NumItems:
      Batch size used for ``get_many`` operations used by the fold
      function

   :type NumItems: integer()

   :param Flags:

      - List of operational flags to apply to the ``fold_table``
        operation, The supported flags are the same as those for
        link:#brick-simple-get-many[brick_simple:get_many()]

      - ``-type flags_list() :: [do_op_flag() | property()]``
      - ``-type do_op_flag() :: 'get_all_attribs' | 'witness'``
        ``{'binary_prefix', binary()} |``
        ``{'max_bytes', integer()}``
      - ``-type property() :: atom() | {term(), term()}``
      - Operational flag usage

        * ``'get_all_attribs'``

          * Return all attributes of each key. May be used in
            combination with the ``witness`` flag

       * ``'witness'``

         * Do not return the value blobs in the result. This flag will
           guarantee that the brick does not require disk access to
           satisfy this request

       * ``{'binary_prefix', binary()}``

         * Return only keys that have a binary prefix that is exactly
           equal to ``binary()``

       * ``{'max_bytes', integer()}``

         * Return only as many keys as the sum of the sizes of their
           corresponding value blobs does not exceed ``integer()``
           bytes

   :type Flags: flags_list()

   :param MaxParallel:

      - If ``MaxParallel`` = 0, a true fold will be performed. If
        ``MaxParallel`` >= 1, then an independent fold will be
        performed on each chain, with up to ``MaxParallel`` number of
        folds running in parallel. The result from each chain fold
        will be returned to the caller as-is, i.e. will **not** be
        combined like in a "reduce" phase of a map-reduce cycle
      - Optional; defaults to 0

   :type MaxParallel: integer()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success return**

   :rtype: ``{ok, Acc::term(), Iterations::integer()}``

   **Error return**

   :rtype: ``{error, Error::term(), Acc::term(), Iterations::integer()}``


Examples
^^^^^^^^

**to be added**

brick_simple:fold_key_prefix/9
------------------------------

For a binary key prefix ``Prefix``, fold over all keys in table
``Table`` starting with ``StartKey``, sleeping for ``SleepTime``
milliseconds between iterations and using ``Flags`` and ``NumItems``
as arguments to link:#brick-simple-get-many[brick_simple:get_many()].
For general information about the Erlang fold function that underlies
this operations, see http://www.erlang.org/doc/man/lists.html#foldl-3.

.. important::
   Do not execute this operation while a data migration is being
   performed

.. erl:function:: fold_key_prefix(Table, Prefix, Fun, Acc, Flags)
.. erl:function:: fold_key_prefix(Table, Prefix, StartKey, Fun, Acc, Flags, NumItems, SleepTime, Timeout)

   :param Table:
      Name of the table in which to perform the fold operation

      - ``-type table() :: atom()``

   :type Table: table()

   :param Prefix: Key prefix for which to perform the fold operation
   :type  Prefix: binary()

   :param StartKey:

      - Key at which to initiate the fold operation
      - Optional; defaults to equal your specified ``Prefix``

   :type StartKey: binary()

   :param Fun:
      Function to apply to successive elements of the list

      - ``-type fun_arity_2() :: fun(({ChainName, TupleFromGetMany}, Acc) -> Acc)``

        * ``TupleFromGetMany`` is a single result tuple from a
          link:#brick-simple-get-many[brick_simple:get_many()]
          result. Its format can vary according to the ``Flags``
          argument, which is passed as-is to a ``get_many()`` call. For
          example, if ``Flags`` = ``[]``, then ``TupleFromGetMany``
          will match ``{Key, TS, Value}``. If ``Flags`` = ``[witness]``,
          then ``TupleFromGetMany`` will match ``{Key, TS}``

      - ``Acc``

        * The accumulator term

   :type Fun: fun_arity_2()

   :param Acc: Initial value of the accumulator term
   :type Acc:  term()

   :param Flags:

      - List of operational flags to apply to the ``fold_key_prefix``
        operation. The supported flags are the same as those for
        link:#brick-simple-get-many[brick_simple:get_many()],
        excluding the ``{'binary_prefix', binary()}`` flag. This flag
        is inappropriate since the key prefix is passed directly
        through the ``Prefix`` argument of
        ``brick_simple:fold_key_prefix()``
      - ``-type flags_list() :: ['get_all_attribs' | 'witness'``
        ``| {'max_bytes', integer()}]``
      - Operational flag usage

          * ``'get_all_attribs'``

            * Return all attributes of each key. May be used in
              combination with the ``witness`` flag

          * ``'witness'``

            * Do not return the value blobs in the result. This flag
              will guarantee that the brick does not require disk
              access to satisfy this request

          * ``{'max_bytes', integer()}``

            * Return only as many keys as the sum of the sizes of
              their corresponding value blobs does not exceed
              ``integer()`` bytes

   :type Flags: flags_list()

   :param NumItems:
      Batch size used for ``get_many`` operations used by the fold
      function

   :type NumItems: integer()

   :param SleepTime:

      - Sleep time between interations, in milliseconds
      - Optional; defaults to 0

   :type SleepTime: integer()

   :param Timeout:

      - Operation timeout in milliseconds
      - **Optional;** defaults to 15000
      - ``-type timeout() :: integer() | 'infinity'``

   :type Timeout: timeout()

   **Success return**

   :rtype: ``{ok, Acc::term(), Iterations::integer()}``

   **Error return**

   :rtype: ``{error, Error::term(), Acc::term(), Iterations::integer()}``

Examples
^^^^^^^^

**to be added**
