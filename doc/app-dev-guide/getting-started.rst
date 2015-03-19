.. highlight:: shell

Getting Started
===============

This section covers the following topics to help you get up and
running with Hibari:

- link:#system-requirements[System Requirements]
- link:#required-software[Required Third Party Software]
- link:#download-hibari[Downloading Hibari]
- link:#installing-single-node[Installing a Single-Node Hibari System]
- link:#starting-single-node[Starting and Stopping a Single-Node Hibari System]
- link:#installing-multi-node[Installing a Multi-Node Hibari Cluster]
- link:#starting-multi-node[Starting and Stopping a Multi-Node Hibari Cluster]
- link:#creating-tables[Creating New Tables]

[[system-requirements]]

System Requirements
-------------------

Hibari will run on any OS that the Erlang VM supports, which includes
most Unix and Unix-like systems, Windows, and Mac OS X. See
`Implementation and Ports of Erlang <http://www.erlang.org/faq/implementations.htm>`_
from the official Erlang documentation for further
information.

For guidance on hardware requirements in a production environment, see
link:hibari-sysadmin-guide.en.html#brick-hardware[Notes on Brick
Hardware] in the Hibari System Administrator's Guide.

[[required-software]]

Required Third-Party Software
-----------------------------

Hibari's requirements for third party software depend on whether
you're doing a single-node installation or a multi-node installation.

Required Software for a Single-Node Installation:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The node on which you plan to install Hibari must have the following software:

- OpenSSL - http://www.openssl.org/

  * Required for Erlang's "crypto" module

Required Software for a Multi-Node Installation:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When you install Hibari on multiple nodes you will use an installer
tool that simplifies the cluster set-up process. When you use this
tool you will identify the hosts on which you want Hibari to be
installed, and the tool will manage the installation of Hibari onto
those target hosts. You can run the tool itself from one of your
target Hibari nodes or from a different machine. There are distinct
requirements for third party software on the "installer node" (the
machine from which you run the installer tool) and on the Hibari nodes
(the machines on which Hibari will be installed and run.)

Installer Node Required Software
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The installer node must have the software listed below. If you are
missing any of these items, you can use the provided links for
downloads and installation instructions.

- Bash - http://www.gnu.org/software/bash/
- Expect - http://www.nist.gov/el/msid/expect.cfm
- Perl - http://www.perl.org/
- SSH (client) - http://www.openssh.com/
- Git - http://git-scm.com/

  * Must be version 1.5.4 or newer
  * If you haven't yet done so, please configure your email address
    and name for Git::

      $ git config --global user.email "you@example.com"
      $ git config --global user.name "Your Name"

 * If you haven't yet done so, you must sign up for a GitHub account -
   https://github.com/

There are currently no known version requirements for Bash, Expect,
Perl, or SSH.


Hibari Nodes Required Software
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The nodes on which you plan to install Hibari must have the software
listed below.

- SSH (server) - http://www.openssh.com/
- OpenSSL - http://www.openssl.org/

  * Required for Erlang's "crypto" module

[[download-hibari]]

Downloading Hibari
------------------

Hibari is not yet available as a pre-built release. In the meanwhile,
you can build Hibari from source. Follow the instructions in
<<HibariBuildingSource>>, and then return to this section to continue
the set-up process.

When you build Hibari your output is two files that you will later use
in the set-up process:

- A tarball package ``hibari-X.Y.Z-DIST-ARCH-WORDSIZE.tgz``
- An md5sum file ``hibari-X.Y.Z-DIST-ARCH-WORDSIZE-md5sum.txt``

``X.Y.Z`` is the release version, ``DIST`` is the release distribution,
``ARCH`` is the release architecture, and ``WORDSIZE`` is the release
wordsize.

[[installing-single-node]]

Installing a Single-Node Hibari System
--------------------------------------

A single-node Hibari system will not provide data replication and
redundancy in the way that a multi-node Hibari cluster will. However,
you may wish to deploy a simple single-node Hibari system for testing
and development purposes.

#. Create a directory for running Hibari::

     $ mkdir running-directory

#. Untar the Hibari tarball package that you created when you built
   Hibari from source::

     $ tar -C running-directory -xvf hibari-X.Y.Z-DIST-ARCH-WORDSIZE.tgz

.. important::
   On your Hibari node, in the system's ``/etc/sysctl.conf`` file,
   set ``vm.swappiness=1``. Swappiness is not desirable for an Erlang VM.


[[starting-single-node]]

Starting and Stopping Hibari on a Single Node
---------------------------------------------

Starting and Bootstrapping Hibari
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Start Hibari::

     $ running-directory/hibari/bin/hibari start

#. If this is the first time you've started Hibari, bootstrap the system::

     $ running-directory/hibari/bin/hibari-admin bootstrap

The Hibari bootstrap process starts Hibari's Admin Server on the
single node and creates a single table "tab1" serving as Hibari's
default table. For information on creating additional tables, see
link:#creating-tables[Creating New Tables].


Verifying Hibari
^^^^^^^^^^^^^^^^

Do these quick checks to verify that your single-node Hibari system is
up and running.

#. Confirm that you can open the "Hibari Web Administration" page::

     $ your-favorite-browser http://127.0.0.1:23080

#. Confirm that you can successfully ping the Hibari node::

     $ running-directory/hibari/bin/hibari ping


IMPORTANT: A single-node Hibari system is hard-coded to listen on the
localhost address 127.0.0.1. Consequently the Hibari node is reachable
only from the node itself.

Stopping Hibari
^^^^^^^^^^^^^^^

To stop Hibari::

  $ running-directory/hibari/bin/hibari stop


[[installing-multi-node]]

Installing a Multi-Node Hibari Cluster
--------------------------------------

Before you install Hibari on to the target nodes you must complete
these preparation steps:

- Set up required user privileges on the installer node and on the
  target Hibari nodes.
- Download the Cluster installer tool.
- Configure the Cluster installer tool.


Setting Up Your User Privileges
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The system user ID that you use to perform the installation must be
different than the Hibari runtime user. Your installing user account
($USER) must be set up as follows:

- $USER must exist on the installer node and also on the target Hibari
  nodes.
- $USER on the installer node must have SSH private/public keys, with
  the SSH agent set up to enable password-less SSH login.
- $USER account must be accessible with password-less SSH login on the
  target Hibari nodes.
- $USER must have password-less sudo access on the target Hibari
  nodes.

If your installing user account does not currently have the above
privileges, follow these steps:

#. As the root user, add your installing user ($USER) to the installer
   node. Then on each of the Hibari nodes, add your installing user and
   grant your user password-less sudo access::

     $ useradd $USER
     $ passwd $USER
     $ visudo
     # append the following line and save it
     $USER  ALL=(ALL)       NOPASSWD: ALL

.. note::
   If you get a "sudo: sorry, you must have a tty to run sudo" error
   while testing sudo, try commenting out following line inside of the
   ``/etc/sudoers`` file::

     $ visudo
     Defaults    requiretty

#. On the installer node, create a new SSH private/public key for your
   installing user::

    $ ssh-keygen
    # enter your password for the private key
    $ eval `ssh-agent`
    $ ssh-add ~/.ssh/id_rsa
    # re-enter your password for the private key

#. On each of the Hibari nodes:

- Append an entry for the installer node to the ``~/.ssh/known_hosts``
  file.
- Append an entry for your public SSH key to the
  ``~/.ssh/authorized_keys`` file.

In the example below, the target Hibari nodes are dev1, dev2, and
dev3::

  $ ssh-copy-id -i ~/.ssh/id_rsa.pub $USER@dev1
  $ ssh-copy-id -i ~/.ssh/id_rsa.pub $USER@dev2
  $ ssh-copy-id -i ~/.ssh/id_rsa.pub $USER@dev3

.. note::
   If your installer node will be one of the Hibari cluster nodes,
   make sure that you ssh-copy-id to the installer node also.

#. Confirm that password-less SSH access to the each of the Hibari
   nodes works as expected::

     $ ssh $USER@dev1
     $ ssh $USER@dev2
     $ ssh $USER@dev3

.. tip::
   If you need more help with SSH set-up, check
   http://inside.mines.edu/~gmurray/HowTo/sshNotes.html.


[[download-cluster]]

Downloading the Cluster Installer Tool
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

"Cluster" is a simple tool for installing, configuring, and
bootstrapping a cluster of Hibari nodes. The tool is not part of the
Hibari package itself, but is available from GitHub.

.. note::
   The Cluster tool should meet the needs of most users.  However,
   this tool's "target node" recipe is currently Linux-centric
   (e.g. useradd, userdel, ...).  Patches and contributions for other OS
   and platforms are welcome.  For non-Linux deployments, the Cluster
   tool is rather simple so installation can be done manually by
   following the tool's recipe.

#. Create a working directory into which you will download the Cluster
   installer tool::

     $ mkdir working-directory

#. Download the Cluster tool's Git repository from GitHub::

     $ cd working-directory
     $ git clone git://github.com/hibari/clus.git

The download creates a sub-directory ``clus`` under which the installer
tool and various supporting files are stored.


[[config-cluster]]

Configuring the Cluster Installer Tool
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Cluster tool requires some basic configuration information that
indicates how you want your Hibari cluster to be set up. You will
create a simple text file that specifies your desired configuration,
and then later use the file as input when you run the Cluster tool.

It's simplest to create the file in the same working directory in
which you downloaded the cluster tool. You can give the file any name
that you want; for purposes of these instructions we will use the file
name ``hibari.config``.

Below is a sample ``hibari.config`` file. The file that you create must
include all of these parameters, and the values must be formatted in
the same way as in this example (with parentheses and quotation marks
as shown). Parameter descriptions follow the example file.

.. code-block:: shell

   ADMIN_NODES=(dev1 dev2 dev3)
   BRICK_NODES=(dev1 dev2 dev3)
   BRICKS_PER_CHAIN=2

   ALL_NODES=(dev1 dev2 dev3)
   ALL_NETA_ADDRS=("10.181.165.230" "10.181.165.231" "10.181.165.232")
   ALL_NETB_ADDRS=("10.181.165.230" "10.181.165.231" "10.181.165.232")
   ALL_NETA_BCAST="10.181.165.255"
   ALL_NETB_BCAST="10.181.165.255"
   ALL_NETA_TIEBREAKER="10.181.165.1"

   ALL_HEART_UDP_PORT="63099"
   ALL_HEART_XMIT_UDP_PORT="63100"


[[eligible-admin-nodes]]

- ``ADMIN_NODES``

  * Host names of the nodes that will be eligible to run the Hibari
    Admin Server. For complete information on the Admin Server, see
    link:hibari-sysadmin-guide.en.html#admin-server-app[The Admin
    Server Application] in the Hibari System Administrator's Guide.

- ``BRICK_NODES``

  * Host names of the nodes that will serve as Hibari storage
    bricks. Note that in the sample configuration file above there are
    three storage brick nodes (dev1, dev2, and dev3), and these three
    nodes are each eligible to run the Admin Server.

- ``BRICKS_PER_CHAIN``

  * Number of bricks per replication chain. For example, with two
    bricks per chain there will be two copies of the data stored in
    the chain (one copy on each brick); with three bricks per chain
    there will be three copies, and so on. For an overview of chain
    replication, see link:#chain-replication[Chain Replication for
    High Availability and Strong Consistency] in this document. For
    chain replication detail, see the Hibari System Administrator's
    Guide.

- ``ALL_NODES``

  * This list of all Hibari nodes is the union of ``ADMIN_NODES`` and
    ``BRICK_NODES``.

- ``ALL_NETA_ADDRS``

  * As described in
    link:hibari-sysadmin-guide.en.html#partition-detector[The
    Partition Detector Application] in the Hibari System
    Administrator's guide, the nodes in a multi-node Hibari cluster
    should be connected by two networks, Network A and Network B, in
    order to detect and manage network partitions. The
    ``ALL_NETA_ADDRS`` parameter specifies the IP addresses of each
    Hibari node within Network A, which is the network through which
    data replication and other Erlang communications will take
    place. The list of the IP addresses should correspond in order to
    host names you listed in the ``ALL_NODES`` setting.

- ``ALL_NETB_ADDRS``

  * IP addresses of each Hibari node within Network B. Network B is
    used only for heartbeat broadcasts that help to detect network
    partitions. The list of the IP addresses should correspond in
    order to host names you listed in the ``ALL_NODES`` setting.

- ``ALL_NETA_BCAST``

  * IP broadcast address for Network A.

- ``ALL_NETB_BCAST``

  * IP broadcast address for Network B.

- ``ALL_NETA_TIEBREAKER``

  * Within Network A, the IP address for the network monitoring
    application to use as a "tiebreaker" in the event of a
    partition. If the network monitoring application on a Hibari node
    determines that Network A is partitioned and Network B is not
    partitioned, then if the Network A tiebreaker IP address responds
    to a ping, then the local node is on the "correct" side of the
    partition. Ideally the tiebreaker should be the address of the
    Layer 2 switch or Layer 3 router that all Erlang network
    distribution communications flow through.

- ``ALL_HEART_UDP_PORT``

  * UDP port for heartbeat listener.

- ``ALL_HEART_XMIT_UDP_PORT``

  * UDP port for heartbeat transmitter.

For more detail on network monitoring configuration settings, see the
partition-detector's OTP application source file
(https://github.com/hibari/partition-detector/raw/master/src/partition_detector.app.src).

CAUTION: In a production setting, Network A and Network B should be
physically different networks and network interfaces.  However, for
testing and development purposes the same physical network can be used
for Network A and Network B (as in the sample configuration file
above).

As final configuration steps, on **each Hibari node**:

- Make sure that the ``/etc/hosts`` file has entries for all Hibari nodes
  in the cluster. For example::

    10.181.165.230  dev1.your-domain.com    dev1
    10.181.165.231  dev2.your-domain.com    dev2
    10.181.165.232  dev3.your-domain.com    dev3

- In the system's ``/etc/sysctl.conf`` file, set ``vm.swappiness=1``. Swappiness
  is not desirable for an Erlang VM.


Installing Hibari
^^^^^^^^^^^^^^^^^

From your installer node, logged in as the installer user, take these
steps to create your Hibari cluster:

#. In the working directory in which you
   link:#download-cluster[downloaded the Cluster tool] and
   link:#config-cluster[created your cluster configuration file], place
   a copy of the Hibari tarball package and md5sum file::

     $ cd working-directory
     $ ls -1
     clus
     hibari-X.Y.Z-DIST-ARCH-WORDSIZE-md5sum.txt
     hibari-X.Y.Z-DIST-ARCH-WORDSIZE.tgz
     hibari.config
     $

#. Create the "hibari" user on all Hibari nodes::

     $ for i in dev1 dev2 dev3 ; do ./clus/priv/clus.sh -f init hibari $i ; done
     hibari@dev1
     hibari@dev2
     hibari@dev3

.. note::
   If the "hibari" user already exists on the target nodes, the -f
   option will forcefully delete and then re-create the "hibari" user.

#. Install the Hibari package on all Hibari nodes, via the newly
   created "hibari" user::

     $ ./clus/priv/clus-hibari.sh -f init hibari hibari.config hibari-X.Y.Z-DIST-ARCH-WORDSIZE.tgz
     hibari@dev1
     hibari@dev2
     hibari@dev3

.. note::
   By default the Cluster tool installs Hibari into
   ``/usr/local/var/lib`` on the target nodes. If you prefer a different
   location, before doing the install open the ``clus.sh`` script (in your
   working directory, under ``/clus/priv/``) and edit the ``CT_HOMEBASEDIR``
   variable.


[[starting-multi-node]]

Starting and Stopping a Multi-Node Hibari Cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can use the Cluster installer tool to start and stop your
multi-node Hibari cluster, working from the same node from which you
managed the installation process. Note that in each of the Hibari
commands in this section you'll be referencing the name of the
link:#config-cluster[Cluster tool configuration file] that you created
during the installation procedure.

Starting and Bootstrapping the Hibari Cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Change to the working directory in which you downloaded the Cluster
   tool, then start Hibari on all Hibari nodes via the "hibari" user::

     $ cd working-directory
     $ ./clus/priv/clus-hibari.sh -f start hibari hibari.config
     hibari@dev1
     hibari@dev2
     hibari@dev3

#. If this is the first time you've started Hibari, bootstrap the
   system via the "hibari" user::

     $ ./clus/priv/clus-hibari.sh -f bootstrap hibari hibari.config
     hibari@dev1 => hibari@dev1 hibari@dev2 hibari@dev3

The Hibari bootstrap process starts Hibari's Admin Server on the first
link:#eligible-admin-nodes[eligible admin node] and creates a single
table "tab1" serving as Hibari's default table. For information about
creating additional tables, see link:#creating-tables[Creating New Tables].

.. note::
   If bootstrapping fails due to "another_admin_server_running"
   error, please stop the other Hibari cluster(s) running on the network;
   or reconfigure the Cluster tool to assign
   link:#eligible-admin-nodes[Hibari heartbeat listener ports] that are
   not in use by another Hibari cluster or other applications and then
   repeat the cluster installation procedure.

Verifying the Hibari Cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Do these simple checks to verify that Hibari is up and running.

#. Confirm that you can open the "Hibari Web Administration" page::

     $ your-favorite-browser http://dev1:23080

#. Confirm that you can successfully ping each of your Hibari nodes::

     $ ./clus/priv/clus-hibari.sh -f ping hibari hibari.config
     hibari@dev1 ... pong
     hibari@dev2 ... pong
     hibari@dev3 ... pong

Stopping the Hibari Cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Stop Hibari on all Hibari nodes via the "hibari" user::

  $ cd working-directory
  $ ./clus/priv/clus-hibari.sh -f stop hibari hibari.config
  ok
  ok
  ok
  hibari@dev1
  hibari@dev2
  hibari@dev3


[[creating-tables]]

Creating New Tables
-------------------

The simplest way to create a new table is via the Admin Server's
GUI. Open ``http://localhost:23080/`` and click the "Add a table" link.
In addition to the GUI, the hibari-admin tool can also be used to
create a new table.  See the hibari-admin tool for usage details.

.. note::
   For information about creating tables using the administrative
   API, see the Hibari System Administrator's Guide.

When adding a table through the GUI, you have these table
configuration options:

- ``Local``

  * Boolean. If true, all bricks for storing the new table's data will
    be created on the local node, i.e. the node that's running the
    Admin Server.  If false, then the "NodeList" field is used to
    specify which cluster nodes the new bricks should use.

- ``BigData``

  * Boolean. If true, value blobs will be stored on disk.

- ``DiskLogging``

  * Boolean. If true, all updates will be written to the write-ahead
    log for persistence.  If false, bricks will run faster but at the
    expense of data loss in a cluster-wide power failure.

- ``SyncWrites``

  * Boolean. If true, all writes to the write-ahead log will be
    flushed to stable storage via the ``fsync(2)`` system call.  If
    false, bricks will run faster but at the expense of data loss in a
    cluster-wide power failure.

- ``VarPrefix``

  * Boolean. If true, then a variable-length prefix of the key will be
    used as input for the consistent hashing function.  If false, the
    entire key will be used.

Many applications can benefit from using a variable-length or
fixed-length prefix hashing scheme.  As an example, consider an
application that maintains state for various users.  The app wishes to
use micro-transactions to update various keys (in the same table)
related to that user.  The table can be created to use
``VarPrefix=true``, together with ``VarPrefixSeparator=47`` (ASCII 47 is
the forward slash character) and ``VarPrefixNumSeparator=2``, to create
a hashing scheme that will guarantee that keys ``/FooUser/summary`` and
``/FooUser/thing1`` and ``/FooUser/thing9`` are all stored by the same
chain.

.. note::
   The HTTP interface for creating tables does not expose the
   fixed-length key prefix scheme.  The Erlang API must be used in this
   case.

- ``VarPrefixSeparator``

  * Integer. Define the character used for variable-length key prefix
    calculation.  Note that the default value of ASCII 47 (the "/"
    character), or any other character, does not imply any UNIX/POSIX
    style file or directory semantics.

- ``VarPrefixNumSeparators``

  * Integer. Define the number of ``VarPrefixSeparator`` bytes, and all
    bytes in between, used for consistent hashing.  If
    ``VarPrefixSeparator=47`` and ``VarPrefixNumSeparators=3``, then for a
    key such as ``/foo/bar/baz``, the prefix used for consistent hashing
    will be ``/foo/bar/``.

- ``Bricks``

  * Integer. If ``Local=true`` (see above), then this integer defines
    the total number of logical bricks that will be created on the
    local node.  This value is ignored if ``Local=false``.

- ``BPC``

  * Integer. Define the number of bricks per chain.

The algorithm used for creating chain -> brick mapping is based on a
"striping" principle: enough chains are laid across bricks in a
stripe-wise manner so that all nodes (aka physical bricks) will have
the same number of logical bricks in head, middle, and tail roles.
See the example in the Hibari System Administrator's Guide of
link:hibari-sysadmin-guide.en.html#3-chains-striped-across-3-bricks[3
chains striped across three nodes].

The Erlang API must be used to create tables with other chain layout
patterns.

- ``NodeList``

  * Comma-separated string. If ``Local=false``, specify the list of
    nodes that will run logical bricks for the new table.  Each node
    in the comma-separated list should take the form
    ``NodeName@HostName``.  For example, use ``hibari1@machine-a,
    hibari1@machine-b, hibari1@machine-c`` to specify three nodes.

- ``NumNodesPerBlock``

  * Integer. If ``Local=false``, then this integer will affect the
    striping behavior of the default chain striping algorithm.  This
    value must be zero (i.e. this parameter is ignored) or a multiple
    of the ``BPC`` parameter.

For example, if ``NodeList`` contains nodes A, B, C, D, E, and F, then
the following striping patterns would be used:

- ``NumNodesPerBlock=0`` would stripe across all 6 nodes for 6
  chains total.
- ``NumNodesPerBlock=2`` and ``BPC=2`` would stripe 2 chains across
  nodes A & B, 2 chains across C & D, and 2 chains across E & F.
- ``NumNodesPerBlock=3`` and ``BPC=3`` would stripe 3 chains across
  nodes A & B & C and 3 chains across D & E & F.

- ``BlockMultFactor``

  * Integer. If ``Local=false``, then this integer will affect the
    striping behavior of the default chain striping algorithm.  This
    value must be zero (i.e. this parameter is ignored) or greater
    than zero.

For example, if ``NodeList`` contains nodes A, B, C, D, E, and F, then
the following striping patterns would be used:

- ``NumNodesPerBlock=0`` and ``BlockMultFactor=0`` would stripe
  across all 6 nodes for 6 chains total.
- ``NumNodesPerBlock=2`` and ``BlockMultFactor=5`` and ``BPC=2`` would
  stripe 2*5=10 chains across nodes A & B, 2*5=10 chains across C
  & D, and 2*5=10 chains across E & F, for a total of 30 chains.
- ``NumNodesPerBlock=3`` and ``BlockMultFactor=4`` and ``BPC=3`` would
  stripe 3*4=12 chains across nodes A & B & C and 3*4=12 chains
  across D & E & F, for a total of 24 chains.
