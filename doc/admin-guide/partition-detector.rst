The Partition Detector Application
==================================

For multi-node Hibari deployments, Hibari includes a network
monitoring feature that watches for partitions within the cluster, and
attempts to minimize the database consequences of such partitions.
This Erlang/OTP application is called the Partition Detector.

You can configure the network monitoring feature in the `central.conf`
file.  See xref:central-conf-parameters[] for details.

IMPORTANT: Use of this feature is mandatory for a multi-node Hibari
deployment to prevent data corruption in the event of a network
partition.  If you don't care about data loss, then as an ancient
Roman might say, ``Caveat emptor.''
Or in English, ``Let the buyer beware.''

For the network monitoring feature to work properly, you must first
set up two separate networks, Network A and Network B, that connect to
each of your Hibari physical bricks. The networks must be set up as
follows:

* Network A and Network B must be physically separate networks, with
different IP and broadcast addresses. See the diagram below for a two
node cluster.
* Network A must be the network used for all Hibari data communications.
* Network A should have as few physical failure points as
possible. For example, a single switch or load balancer is preferable
to two switches cabled together.
* The separate Network B will be used to compare node heartbeat patterns.

IMPORTANT: For the network partition monitor to work properly, your
network partition monitor configuration settings must match as closely
as possible.  Each Hibari physical brick must have unique IP addresses
on its two network interfaces (as required by all IP networks), but
all configurations must use the same IP subnets for the 'A' and 'B'
networks, and all configurations must use the same network 'A'
tiebreaker.

[[a-and-b-network-diagram]]
.Network 'A' and network 'B' diagram
svgimage::images/a-and-b-diagram[align="center", scaledwidth="80%"]

=== Partition Detector Heartbeats

Through the partition monitoring application, Hibari nodes send
heartbeat messages to one another at the configurable
heartbeat_beacon_interval, and each node keeps track of heartbeat
history from each of the other nodes in the cluster. The heartbeats
are transmitted through both Network A and Network B. If node
`gdss1@machine1` detects that the incoming heartbeats from
`gdss1@machine2` are absent both on Network A and on Network B, then
`gdss1@machine2` might have a problem. If the incoming heartbeats from
`gdss1@machine2` fail on Network A but not on Network B, a partition on
Network A might be the cause. If heartbeats fail on Network B but not
Network A, then Network B might have a partition problem, but this is
less serious because Hibari data communication does not take place on
Network B.

Configurable timers on each Hibari node determine the interval at
which the absence of incoming heartbeats from another node is
considered a problem. If on node `gdss1@machine1` no heartbeat has been
received from `gdss1@machine2` for the duration of the configurable
`heartbeat_warning_interval`, then a warning message is
written to the application log of node `gdss1@machine1`. This warning
message can be triggered by missing heartbeats either on Network A or
on Network B; the warning message will indicate which node has not
been heard from, and over which network.

=== Partition Detector's Tiebreaker

If on node `gdss1@machine1` no heartbeat has been received from
`gdss1@machine2` via Network A for the duration of the configurable
`heartbeat_failure_interval`, and if during that period heartbeats
from `gdss1@machine2` continue to be received via Network B, then a
network partition is presumed to have occurred in Network A. In this
scenario, node `gdss1@machine1` will attempt to ping the configurable
`network_a_tiebreaker` address. If `gdss1@machine1` successfully pings
the tiebreaker address, then `gdss1@machine1` considers itself to be
on the "correct" side of the Network A partition, and it continues
running. If by contrast `gdss1@machine1` cannot successfully ping the
tiebreaker address, then `gdss1@machine1` considers itself to be on
the "wrong" side of the Network A partition and shuts itself
down. Meanwhile, comparable calculations and decisions are being made
by node `gdss1@machine2`.

In a scenario where the network monitoring application determines that
a partition has occurred on Network B -- that is, heartbeats are received
through Network A but not through Network B -- then warnings are written
to the Hibari nodes' application logs but no node is shut down.
