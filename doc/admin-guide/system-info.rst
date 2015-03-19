Hibari System Information: Configuration Files, Etc.
====================================================

Hibari's system information is stored in one of two places.  The first
is the application configuration file, `central.conf`.  By default,
this file is stored in `TODO/{version number}/etc/central.conf`.

The second location is within Hibari server nodes themselves.  This
kind of configuration, stored inside the "bootstrap" bricks, makes it
easy to share data with all nodes in the cluster.

Many of configuration values in `central.conf` will be the same on all
nodes in a Hibari cluster.  Given this reality, why not store those
items in Hibari itself?  The biggest problem comes when the
application is first starting.  See
xref:bricks-outside-chain-replication[] for an overview of why it
isn't easy to store all configuration data inside Hibari itself.

In the future, it's likely that many of the configuration items in the
`central.conf` file will move to storage within Hibari itself.

=== `central.conf` File Syntax and Usage

Each line of the `central.conf` file has the form

  parameter: value

where `parameter` is the name of the configuration option being set and
`value` is the value that the configuration option is being set to.

Valid data types for configuration settings are INT (integer), STRING
(string), and ATOM (one of a pre-defined set of option names, such as
`on` or `off`). Apart from data type restrictions, no further valid
range restrictions are enforced for `central.conf` parameters.

All time values in `central.conf` (such as delivery retry intervals or
transaction timeouts) must be set as a number of seconds.

Blank lines and lines beginning with the pound sign (#) are ignored.

IMPORTANT: To apply changes that you have made to the `central.conf`
file, you must restart the server.  There are exceptions to this rule,
but it's one of the cleanup/janitor tasks to access
`central.conf` using a standard set of APIs, e.g. always use the
`gmt_config_svr` API.

[[central-conf-parameters]]
=== Parameters in the `central.conf` File

A detailed explanation of each of the items in `central.conf` can be
found at
link:../misc-files/central-conf.pdf[Hibari `central.conf` Configuration Guide].

=== Admin Server Configuration

Configuration for the Hibari ``Admin Server'' is stored in three
places:

. The `central.conf` file
. The `Schema.local` file
. Inside the ``bootstrap'' bricks

[[admin-server-in-central-conf]]
==== Admin Server entries in the `central.conf` file

The following entries in the `central.conf` file are used by the
Hibari Admin Server:

* `admin_server_distributed_nodes`
** This option specifies which nodes in the Hibari cluster are
   eligible to run the Admin Server.  Hibari server nodes not included
   in this list cannot run the Admin Server.
** Active/standby service is provided by the Erlang/OTP platform's
   application management facility.
* The `Schema.local` file
** This file provides a list of {logical brick, Hibari server node name}
   tuples that store the Admin Server's private state.  Each brick
   name in this list starts with the prefix `bootstrap_copy` followed
   by an integer.
* The ``bootstrap'' bricks
** Each of these bricks store an independent copy of all Hibari
   cluster state: table definitions, table -> chain mappings,
   start & stop history, etc.
** Data in each of the bootstrap bricks is not maintained by chain
   replication.  Rather, quorum-style replication is used.
   See xref:bricks-outside-chain-replication[].

=== Configuration Not Stored in Editable Config Files

All table and chain configuration parameters are stored within the
Admin Server's ``schema''.  The schema contains information on:

* Table names and options (e.g. blob values stored in RAM or on disk,
  sync/async disk logging)
* Table -> chain mappings
* Chain -> brick mappings

Much of this information can be seen in HTML form by pointing a Web
browser at TCP port 23080 (default) of any Hibari server node.  For
example:

.Admin Server Top-Level Status & Admin URL
    http://hibari-server-node-hostname:23080/

Your Web browser should be redirected automatically to the Admin
Server's top-level status & admin page.

NOTE: The APIs that expose this are, for the most part, already
written.  We need more "friendly" wrapper funcs as part of the "try
this first" set of APIs for administration.
