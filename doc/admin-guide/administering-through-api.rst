Administering Hibari Through the API
====================================

* Add a new table
* Delete a table
* Change to a single chain:
** Add one or more bricks (increase replication factor)
** Remove one or more bricks (decrease replication factor)
* Change to a single table.
** Add a new chain
** Remove a chain
** Change the chain weighting factor
** Change consistent hashing parameters

[[add-a-new-table]]
=== Add a New Table: brick_admin:add_table()

[[why-use-hash-prefixes]]
==== Why use hash prefixes?

Hash prefixes allow Hibari servers to guarantee the application
developer that certain keys will always be stored on the same chain
and therefore always on the same set of bricks.  With this guarantee,
an application aware of hash prefixes can use micro-transactions
successfully.

For example, assume the application requires a collection of
persistent stacks that are stored in Hibari.

* Each stack is identified by a string/binary.  (The two types are
  identical for the sake of discussion.)
* Each item stored on the stack is a string.
* Support stack options push & pop.
* Support quick stack stats, e.g. # of elements on the stack and # of
  bytes stored on the stack.
* Stacks may contain hundreds of thousands of items.
* The total size of a stack will not exceed the total storage capacity
  of any single brick in the cluster.

IMPORTANT: Understanding the last assumption is vital.  Because all
keys with the same hash prefix `H` will be managed by the same chain
`C`, then all bricks in `C` must have enough capacity to store all `H`
prefix keys.

The application developer then makes the following decisions:

1. The application will use a table devoted to storing stacks, called
   `'stack'`.
2. We know that the application requires strong durability (which is
   the Hibari default) and that the sum total of all stack *items* will
   exceed a single brick's RAM capacity.  Therefore, the `'stack'`
   table must store its value blobs on disk.  Read access to the table
   will be slower than if value blobs were stored in RAM, but the
   limited RAM capacity of bricks does not give us a choice.
3. We have two machines, `boxA` and `boxB`, available for hosting the
   table's logical bricks.
   We want to be able to survive at least one physical brick failure,
   therefore all chains have a minimum length of 2.
** We will use two chains, so that each physical machine (when up and
   running smoothly) will have 2 logical bricks for the table, one in
   the chain head role and one in the chain tail role.
** The naming scheme used for each chain name and brick name can be
   arbitrary, as long as all names are unique.  However, for
   ease-of-management purposes, the use of a systematic naming scheme
   is strongly encouraged.  The scheme used here numbers each chain
   (starting at 1) and numbers each brick (also starting at 1) with
   both the chain and brick number.
4. We use the following key naming convention:
** A stack's metadata (item count, byte count) uses `<<"/StackName/md">>`.
** A item uses `<<"/StackName/N">>` where N is the item number.
5. We create the table using the following:
+
------------------------
Opts = [{hash_init, fun brick_admin:chash_init/3}, {prefix_method, var_prefix},
        {num_separators, 2}, {prefix_separator, $/},
        {new_chainweights, [{stack_ch1, 100}, {stack_ch2, 100}]},
        {bigdata_dir, "."}, {do_logging, true}, {do_sync, true}].

ChainList = [{stack_ch1, [{stack_ch1_b1, hibari1@boxA},
                          {stack_ch1_b2, hibari1@boxB}]},
             {stack_ch1, [{stack_ch2_b1, hibari1@boxB},
                          {stack_ch2_b2, hibari1@boxA}]}].

brick_admin:add_table(stack, ChainList, Opts).
------------------------

See xref:examples-using-the-stack[] for sample usage code.

[[types-of-brick-admin-add-table]]
==== Types for brick_admin:add_table()

----------------------------
add_table(Name, ChainList)
  equivalent to add_table(brick_admin, Name, ChainList)

add_table(Name, ChainList, BrickOptions)
  when is_atom(Name), is_list(ChainList)
  equivalent to add_table(brick_admin, Name, ChainList, BrickOptions)

add_table(ServerRef, Name, BrickOptions)
  when is_atom(Name), is_list(BrickOptions)
  equivalent to add_table(ServerRef, Name, ChainList, [])

add_table(ServerRef::gen_server_serverref(), Name::table(),
          ChainList::chain_list(), BrickOptions::brick_options())
-> ok |
   {error, term()} |
   {error, term(), term()}

gen_server_serverref() = "ServerRef" type from STDLIB gen_server, gen_fsm, etc.
proplists_property()   = "Property" type from STDLIB proplists

bigdata_option()    = {'bigdata_dir', string()}
brick()             = {logical_brick(), node()}
brick_option()      = chash_prop() |
                      custom_prop() |
                      fixed_prefix_prop() |
                      {'hash_init', fun/3} |
                      var_prefix_prop()
brick_options()     = [brick_option]
chain_list()        = {chain_name(), [brick()]}
chain_name()        = atom()
chash_prop()        = {'new_chainweights', chain_weights()} |
                      {'num_separators', integer()} |
                      {'old_float_map', float_map()} |
                      {'prefix_is_integer_hack', boolean()} |
                      {'prefix_length', integer()} |
                      {'prefix_method', 'all' | 'var_prefix' | 'fixed_prefix'} |
                      {'prefix_separator', integer()}
chain_weights()     = [{chain_name, integer()}]
custom_prop()       = proplists_property()
fixed_prefix_prop() = {'prefix_is_integer_hack', boolean()} |
                      {'prefix_length', integer()}
logging_option()    = {'do_logging', boolean()}
logical_brick()     = atom()
node()              = atom()
sync_option()       = {'do_sync', boolean()}
table()             = atom()
var_prefix_prop()   = {'num_separators', integer()} |
                      {'prefix_separator', integer()}
----------------------------

`{'bigdata_dir', string()}`::
To store value blobs on disk (i.e. "big data" is true), specify this
value with any string (the string's actual value is not used).
+
IMPORTANT: To store value blobs in RAM, this option must be omitted.
+
`{'do_logging', boolean()}`::
Specify whether all bricks in the table will log updates to disk.
If not specified, the default is true.
+
`{'do_sync', boolean()}`::
Specify whether all bricks in the table will synchronously flush all
updates to disk before responding to the client.
If not specified, the default is true.
+
`{'hash_init', fun/3}`::
Specify the hash initialization function.  Of the four hash methods
bundled with Hibari, we recommend using `brick_hash:chash_init/3`
only.
+
`{'new_chainweights, chain_weights()}`::
(For `brick_admin:chash_init/3`)
Specify the chainweights for this new
table.  For creating a new table, this option is not used.
However, this option is used when changing a table to
add/remove chains or to change other table-related parameters.
+
`{'num_separators', integer()}`::
(For `brick_admin:chash_init/3` and `brick_admin:var_prefix_init/3`)
For variable prefix hashes, this option specifies how many instances
of the variable prefix separator character (see `'prefix_separator'`
below) are included in the hashing prefix.
The default is 2.
+
For example, if `{'prefix_separator', $/}`, then
+
** With `{'num_separators', 2}` and key `<<"/foo/bar/baz/hello">>`,
   the hashing prefix is `<<"/foo/">>`.
** With `{'num_separators', 3}` and key `<<"/foo/bar/baz/hello">>`,
   the hashing prefix is `<<"/foo/bar/">>`.
+
`{'old_float_map', float_map()}`::
Specify the old version of the "float map".
For creating a new table, this option is not used.
However, this option is used when changing a table to
add/remove chains or to change other table-related parameters: it
is used to create a new mapping of {table, key} -> chain that
relocates only a minimum number of keys a new chain.
+
`{'prefix_method', 'all' | 'var_prefix' | 'fixed_prefix'}`::
(For `brick_admin:chash_init/3`) Specify which prefix method will be
used for consistent hashing:
+
** `'all'`: Use the entire key
** `'var_prefix'`: Use a variable-length prefix of the key
** `fixed_prefix'`: Use a fixed-length prefix of the key
+
`{'prefix_is_integer_hack', boolean()}`::
(For `brick_admin:fixed_prefix_init/3`)
If true, the prefix should be interpreted
as an ASCII representation of a base 10 integer for use as the
hash calculation.
+
`{'prefix_length', integer()}`::
(For `brick_admin:fixed_prefix_init/3`)
For a fixed-prefix hashes, this option specifies the prefix length.
+
`{'prefix_separator', integer()}`::
(For `brick_admin:chash_init/3` and `brick_admin:var_prefix_init/3`)
For variable prefix hashes, this option specifies the
single byte ASCII value of the byte
that separates the key's prefix from the rest of the key.
The default is $/, ASCII 47.

[[examples-using-the-stack]]
==== Examples code for using the stack

.Create a new stack
------------------------------------------------
Val = #stack_md{count = 0, bytes = 0}.
brick_simple:add(stack, "/new-stack/md", term_to_binary(Val)).
------------------------------------------------

.Push an item onto a stack
------------------------------------------------
{ok, OldTS, OldVal} = brick_simple:get(stack, "/new-stack/md").
#stack_md{count = Count, bytes = Bytes} = binary_to_term(OldVal).
NewMD = #stack_md{count = Count + 1, bytes = Bytes + size(NewItem)}.
ItemKey = "/new-stack/" ++ integer_to_list(Count).
[ok, ok] = brick_simple:do(stack,
                           [brick_server:make_txn(),
                            brick_server:make_replace("/new-stack/md",
                                                      term_to_binary(NewMD),
                                                      0, [{testset, OldTS}]),
                            brick_server:make_add(ItemKey, NewItem)]).
------------------------------------------------

.Pop an item off a stack
------------------------------------------------
{ok, OldTS, OldVal} = brick_simple:get(stack, "/new-stack/md").
#stack_md{count = Count, bytes = Bytes} = binary_to_term(OldVal).
ItemKey = "/new-stack/" ++ integer_to_list(Count - 1).
{ok, _, Item} = brick_simple:get(stack, ItemKey).
NumBytes = proplists:get_value(val_len, Ps).
NewMD = #stack_md{count = Count - 1, bytes = Bytes - size(Item)}.
[ok, ok] = brick_simple:do(stack,
                           [brick_server:make_txn(),
                            brick_server:make_replace("/new-stack/md",
                                                      term_to_binary(NewMD),
                                                      0, [{testset, OldTS}]),
                            brick_server:make_delete(ItemKey)]).
Item.
------------------------------------------------

[[delete-a-table]]
=== Delete a Table

As yet, Hibari does not have a method to delete a table.  The only
methods available now are:

* Delete all files and subdirectories from the `bootstrap_*` brick
  data directories, restart the Admin Server, and recreate all tables.
  (Also known as, "Start over".)
* Make a backup copy of all `bootstrap_*` brick data directories
  before creating a new table.  If you wish to undo, then stop Hibari
  on all Admin Server-eligible nodes, remove the `bootstrap_*` brick
  data directories, restore the `bootstrap_*` brick data directories
  from the previous backup, then start all of the Admin
  Server-eligible nodes.

[[change-a-chain-add-remove-bricks]]
=== Change a Chain: Add or Remove Bricks

Adding or removing bricks from a single chain changes the replication
factor for the keys stored in that chain: more bricks increases the
replication factor, and fewer bricks decreases it.

.Data types for brick_admin:change_chain_length()
--------------------------------------
brick_admin:change_chain_length(ChainName, BrickList)

ChainName       = atom()
BrickList       = [brick()]

brick()         = {logical_brick(), node()}
logical_brick() = atom()
node()          = atom()
--------------------------------------

See also,
xref:example-change-chain-length[`brick_admin:change_chain_length()` usage examples].

[[change-a-table-add-remove-chains]]
=== Change a Table: Add/Remove Chains

.Data types for brick_admin:start_migration()
--------------------------------------
brick_admin:start_migration(TableName, LH)
  equivalent to brick_admin:start_migration(TableName, LH, [])

brick_admin:start_migration(TableName, LH, Options)
-> {ok, cookie()} |
   {'EXIT', term()}

TableName           = atom()
LH                  = hash_r()
Options             = migration_options()

cookie()            = term()
migration_option()  = {'do_not_initiate_serial_ack', boolean()} |
                      {'interval', integer()} |
                      {'max_keys_per_chain', integer()} |
                      {'max_keys_per_iter', integer()} |
                      {'propagation_delay', integer()}
migration_options() = [migration_option()]

brick_admin:chash_init('via_proplist', ChainList, Options)
-> hash_r()

ChainList = chain_list()
Options   = brick_options()
--------------------------------------

See xref:types-of-brick-admin-add-table[] for definitions of
`chain_list()` and `brick_options()` types.

The `hash_r()` type is an Erlang record, `#hash_r` as defined in the
`brick_hash.hrl` header file.  It is normally considered an opaque
type that is created by a function such as `brick_hash:chash_init/3`.

NOTE: The options list passed in argument #3 to
`brick_admin:chash_init/3` is the same properties list that is used
for `brick_admin:add_table/3`.  The difference is that the options
that are related strictly to brick behavior, such as the `do_logging`
and `do_sync` properties, are ignored by `chash_init/3`.

Once a `hash_r()` term is created and `brick_admin:start_migration/2`
is called successfully, the data migration will start immediately.

The `cookie()` type is an opaque term that uniquely identifies the
data migration that was triggered for the `TableName` table.  Another
data migration may not be triggered until the current migration has
finished successfully.

The `migration_option()` properties are described below:

`{'do_not_initiate_serial_ack', boolean()}`::
For internal use only, do not use.
+
`{'interval', integer()}`::
Interval (in milliseconds) to send kick_next_sweep messages.
Default = 50.
+
`{'max_keys_per_chain', integer()}`::
Maximum number of keys
to send to any particular chain.  Not yet implemented.
+
`{'max_keys_per_iter', integer()}`::
Maximum number of keys to examine per sweep iteration.
Default = 500 for bricks with value blobs in RAM, 25 for bricks with
value blobs on disk.
+
`{'propagation_delay', integer()}`::
Number of milliseconds to delay for each brick's logging operation.
Default = 0.

See also xref:changing-chains-example[].

[[change-a-table-chain-chain-weighting]]
=== Change a Table: Change Chain Weighting

The functions to change chain weighting are the same for
adding/removing chains, see xref:change-a-table-add-remove-chains[]
for additional details.

When creating a `hash_r()` type record, follow these two bits of
advice:

* The `chain_list()` term remains exactly the same as the chain list
  currently used by the table.  See
  `brick_admin:get_table_chain_list/1` for how to retrieve this list.
* The `new_chainweights` property in the `brick_options()` list
  specifies a different set of chain weighting factors than is
  currently used by the table.  The current chain weighting list is in
  the `brick_options` property returned by the
  `brick_admin:get_table_info/1` function.

See also xref:changing-chains-example[].

[[admin-server-api]]
=== Admin Server API

See EDoc documentation for `brick_admin.erl` API.

[[scoreboard-api]]
=== Scoreboard API

See EDoc documentation for `brick_sb.erl` API.

[[chain-monitor-api]]
=== Chain Monitor API

See EDoc documentation for `brick_chainmon.erl` API.

[[changing-chain-length]]
=== Changing Chain Length: Examples

The Admin Server's basic definition of a chain: the chains name, and
the list of bricks.  In turn, each brick is defined by a 2-tuple of
brick name and node name.

------------
Chain definition = {ChainName, [BrickDef1, BrickDef2, ...]}
Brick definition = {BrickName, NodeName}

Example chain definition, chain length=1
    {tab1_ch1, [{tab1_ch1_b1, hibari1@bb3}]}
------------

The function `brick_admin:get_table_chain_list/1` will retrieve the
active chain definition list for a table.  For example, we retrieve the
chain definition list for the table `tab1`.  The node `bb3` is the
hostname of my laptop.

----------
(hibari1@bb3)23> {ok, Tab1ChList} = brick_admin:get_table_chain_list(tab1).
{ok,[{tab1_ch1,[{tab1_ch1_b1,hibari1@bb3}]}]}

(hibari1@bb3)24> Tab1ChList.
[{tab1_ch1,[{tab1_ch1_b1,hibari1@bb3}]}]
----------

NOTE: The `brick_admin:get_table_chain_list/1` function will retrieve
the active chain definition list for a table: only bricks that are in
`ok` state will be shown.  If a chain has a brick that has crashed,
that brick will not appear in the list returned by this function.  The
`brick_admin:get_table_info()` function can fetch the list of all
bricks, in service and crashed, but the API is not as convenient.

[[example-change-chain-length]]
To change the chain length, use the
`brick_admin:change_chain_length/2` function.  The arguments are the
chain name and brick list.

NOTE: Any bricks in the brick list that aren't in the chain are
automatically started. Any bricks in the current chain that are not in
the new list are halted, *and their persistent data will be deleted*.

// JWN: The deletion is not immediate on disk - correct?  Scavenger is
// needed - right?

----------------
(hibari1@bb3)29> brick_admin:change_chain_length(tab1_ch1,
                 [{tab1_ch1_b1,hibari1@bb3}, {tab1_ch1_b2,hibari1@bb3}]).
ok

(hibari1@bb3)30> {ok, Tab1ChList2} = brick_admin:get_table_chain_list(tab1). {ok,[{tab1_ch1,[{tab1_ch1_b1,hibari1@bb3},
                {tab1_ch1_b2,hibari1@bb3}]}]}

----------------

Now the `tab1_ch1` chain has length two.  We'll shorten it back down
to length 1.

----------------
(hibari1@bb3)31> brick_admin:change_chain_length(tab1_ch1, [{tab1_ch1_b2,hibari1@bb3}]).
ok

(hibari1@bb3)32> {ok, Tab1ChList3} = brick_admin:get_table_chain_list(tab1).
{ok,[{tab1_ch1,[{tab1_ch1_b2,hibari1@bb3}]}]}
----------------

NOTE: A chain's new brick list must contain at least one brick from
the current chain's definition.  If the intersection of old brick list
and new brick list is empty, the command will fail.

-------------
(hibari1@bb3)34> brick_admin:change_chain_length(tab1_ch1, [{tab1_ch1_b3,hibari1@bb3}]).
{'EXIT',{error,no_intersection}}
-------------

[[changing-chains-example]]
=== Creating and Rebalancing Chains: Examples

The procedure for creating new chains, deleting existing chains, and
reweighing existing chains, and rehashing is done using the the
`brick_admin:start_migration()` function.  The chain definitions are
specified in the same way as changing chain lengths, see
xref:changing-chain-length[] for details.

The data structure required by `brick_admin:start_migration/2` is more
complex than the relatively-simple brick list that
`brick_admin:change_chain_length/2` requires.  This section will
demonstrate the creation of this structure, the ``local hash record'',
step-by-step.

First, we create a new chain definition list.  (Refer to
xref:changing-chain-length[] if necessary.)  For this example, we'll
assume that we'll be modifying the `tab1` table and that we'll be
adding two more chains.  Each chain will be of length one.  We'll
place each chain on the same node as everything else, `hibari1@bb3`
(i.e. my laptop).

---------------
(hibari1@bb3)48> brick_admin:get_table_chain_list(tab1).
{ok,[{tab1_ch1,[{tab1_ch1_b1,hibari1@bb3}]}]}

(hibari1@bb3)49> NewCL = [{tab1_ch1, [{tab1_ch1_b1, hibari1@bb3}]},
                 {tab1_ch2, [{tab1_ch2_b1, hibari1@bb3}]},
                 {tab1_ch3, [{tab1_ch3_b1, hibari1@bb3}]}].
[{tab1_ch1,[{tab1_ch1_b1,hibari1@bb3}]},
 {tab1_ch2,[{tab1_ch2_b1,hibari1@bb3}]},
 {tab1_ch3,[{tab1_ch3_b1,hibari1@bb3}]}]

---------------

NOTE: Any bricks in the brick list that aren't in a chain are
automatically started.  Any bricks in a current chains that are not in
the chain definition are halted, *and their persistent data will be
deleted*.

Next, we retrieve the table's current hashing configuration.  The data
is returned to us in the form of an Erlang property list.  (See the
Erlang/OTP documentation for the `proplists` module, located in the
"Basic Applications" area under "stdlib".)  We then pick out several
properties that we'll need later; we use `lists:keyfind/3` instead of
a function in the `proplists` module because it will preserve the
properties in 2-tuple form, which will save us some typing effort
later.

---------------
(hibari1@bb3)51> {ok, TabInfo} = brick_admin:get_table_info(tab1).
{ok,[{name,tab1},
    ...lots of stuff omitted...

(hibari1@bb3)53> Opts = proplists:get_value(brick_options, TabInfo).
[{hash_init,#Fun<brick_hash.chash_init.3>},
 {old_float_map,[]},
 {new_chainweights,[{tab1_ch1,100}]},
 {hash_init,#Fun<brick_hash.chash_init.3>},
 {prefix_method,var_prefix},
 {prefix_separator,47},
 {num_separators,3},
 {bigdata_dir,"cwd"},
 {do_logging,true},
 {do_sync,true},
 {created_date,{2010,4,17}},
 {created_time,{17,21,58}}]

(hibari1@bb3)58> PrefixMethod = lists:keyfind(prefix_method, 1, Opts).
{prefix_method,var_prefix}

(hibari1@bb3)59> NumSep = lists:keyfind(num_separators, 1, Opts).
{num_separators,3}

(hibari1@bb3)60> PrefixSep = lists:keyfind(prefix_separator, 1, Opts).
{prefix_separator,47}

(hibari1@bb3)61> OldCWs = proplists:get_value(new_chainweights, Opts).
[{tab1_ch1,100}]

(hibari1@bb3)62> OldGH = proplists:get_value(ghash, TabInfo).

(hibari1@bb3)63> OldFloatMap = brick_hash:chash_extract_new_float_map(OldGH).
---------------

Next, we create a new property list.

---------------
(hibari1@bb3)71> NewCWs = OldCWs ++ [{tab1_ch2, 100}, {tab1_ch3, 100}].
[{tab1_ch1,100},{tab1_ch2,100},{tab1_ch3,100}]

(hibari1@bb3)72> NewOpts = [PrefixMethod, NumSep, PrefixSep,
                               {new_chainweights, NewCWs},
                               {old_float_map, OldFloatMap}].
[{prefix_method,var_prefix},
 {num_separators,3},
 {prefix_separator,47},
 {new_chainweights,[{tab1_ch1,100},
                    {tab1_ch2,100},
                    {tab1_ch3,100}]}
 {old_float_map, []}]
---------------

Next, we use the chain definition list, `NewCL`, and the table options
list, `NewOpts`, to create a ``local hash'' record.  This record will
contain all of the configuration information required to change a
table's consistent hashing characteristics.

---------------
(hibari1@bb3)73> NewLH = brick_hash:chash_init(via_proplist, NewCL, NewOpts).
{hash_r,chash,brick_hash,chash_key_to_chain,
 ...lots of stuff omitted...
---------------

[[chash-migration-pre-check]]
We're just one step away from changing the `tab1` table.  Before we
change the table, however, we'd like to see how the table change will
affect the data in the table.  First, we add 1,000 keys to the `tab1`
table.  Then we use the `brick_simple:chash_migration_pre_check/2`
function to tell us how many keys will move and to where.

---------------
(hibari1@bb3)74> [brick_simple:set(tab1, "foo"++integer_to_list(X), "bar") || X <- lists:seq(1,1000)].
[ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,ok,
 ok,ok,ok,ok,ok,ok,ok,ok,ok,ok|...]

(hibari1@bb3)75> brick_simple:chash_migration_pre_check(tab1, NewLH).
[{keys_before,[{tab1_ch1,1001}]},
 {keys_keep,[{tab1_ch1,348}]},
 {keys_moving,[{tab1_ch2,315},{tab1_ch3,338}]},
 {keys_moving_where,[{tab1_ch1,[{tab1_ch2,315},
                                {tab1_ch3,338}]}]},
 {errors,[]}]
---------------

The output above shows us that of the 1,001 keys in the `tab1` table,
348 will remain in the `tab1_ch1` chain, 315 keys will move to the
`tab1_ch2` chain, and 338 keys will move to the `tab1_ch3` chain.
That looks like what we want, so let's reconfigure the table and start
the data migration.

---------------
brick_admin:start_migration(tab1, NewLH).
---------------

Immediately, we'll see a bunch of application messages sent to the
console as new activities start:

* A migration monitoring process is started.
* New brick processes are started.
* New monitoring processes are started.
* Data migrations are started and finish
* The migration monitoring process exits.

---------------
=GMT INFO REPORT==== 20-Apr-2010::00:26:40 ===
Migration number 1 is starting with cookie {1271,741200,988900}

=GMT INFO REPORT==== 20-Apr-2010::00:26:41 ===
progress: [{supervisor,{local,brick_mon_sup}},
           {started,
               [{pid,<0.2937.0>},
                {name,chmon_tab1_ch2},
                ...stuff omitted...

[...lines skipped...]
=GMT INFO REPORT==== 20-Apr-2010::00:26:41 ===
Migration monitor: tab1: chains starting

[...lines skipped...]
=GMT INFO REPORT==== 20-Apr-2010::00:26:41 ===
brick_admin: handle_cast: chain tab1_ch2 in unknown state

[...lines skipped...]
=GMT INFO REPORT==== 20-Apr-2010::00:26:52 ===
Migration monitor: tab1: sweeps starting

[...lines skipped...]
=GMT INFO REPORT==== 20-Apr-2010::00:26:54 ===
Migration number 1 finished

[...lines skipped...]
=GMT INFO REPORT==== 20-Apr-2010::00:26:57 ===
Clearing final migration state for table tab1
---------------

For the sake of demonstration, now let's see what
`brick_simple:chash_migration_pre_check()` would say if we were to
migrate from three chains to four chains.

---------------
(hibari_dev@bb3)24> {ok, TabInfo3} = brick_admin:get_table_info(tab1).

(hibari_dev@bb3)25> Opts3 = proplists:get_value(brick_options, TabInfo3).

(hibari_dev@bb3)26> GH3 = proplists:get_value(ghash, TabInfo3).

(hibari_dev@bb3)28> OldFloatMap = brick_hash:chash_extract_new_float_map(GH3).

(hibari_dev@bb3)31> NewOpts4 = [PrefixMethod, NumSep, PrefixSep,
                    {new_chainweights, NewCWs4}, {old_float_map, OldFloatMap}].

(hibari_dev@bb3)35> NewCL4 = [ {tab1_ch1, [{tab1_ch1_b1, hibari1@bb3}]},
                               {tab1_ch2, [{tab1_ch2_b1, hibari1@bb3}]},
                               {tab1_ch3, [{tab1_ch3_b1, hibari1@bb3}]},
                               {tab1_ch4, [{tab1_ch4_b1, hibari1@bb3}]} ].
(hibari_dev@bb3)36> NewLH4 = brick_hash:chash_init(via_proplist, NewCL4, NewOpts4).

(hibari_dev@bb3)37> brick_simple:chash_migration_pre_check(tab1, NewLH4).
[{keys_before,[{tab1_ch1,349},
               {tab1_ch2,315},
               {tab1_ch3,337}]},
 {keys_keep,[{tab1_ch1,250},{tab1_ch2,232},{tab1_ch3,232}]},
 {keys_moving,[{tab1_ch4,287}]},
 {keys_moving_where,[{tab1_ch1,[{tab1_ch4,99}]},
                     {tab1_ch2,[{tab1_ch4,83}]},
                     {tab1_ch3,[{tab1_ch4,105}]}]},
 {errors,[]}]
---------------

The output tells us that chain `tab1_ch1` will lose 99 keys,
`tab1_ch2` will lose 83 keys, and `tab1_ch3` will lose 105 keys.  The
final key distribution across the four chains would be 250, 232, 232,
and 287 keys, respectively.
