<!--
Copyright (c) 2007-2020 VMware, Inc. or its affiliates.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Stream Queues NOSYNTAX

## <a id="overview" class="anchor" href="#overview">Overview</a>

A new persistent and replicated RabbitMQ queue type modelling an append-only log
with non-destructive consumer semantics. Can be used as a regular AMQP 0.9.1 queue
or through a new binary protocol plugin and associated client(s).

### <a id="use-cases" class="anchor" href="#use-cases">Use Cases</a>

The stream queue type was developed to initially cover 4 messaging use-cases that
existing queue types either can not provide or provide with downsides:

1. Large fan-outs

    When wanting to deliver the same message to multiple subscribers users currently
    have to bind a dedicated queue for each consumer. If the number of consumers is
    large this becomes potentially inefficient, especially when wanting persistence
    and/or replication. The stream queue will allow any number of consumers to consume
    the same messages from the same queue in a non-destructive manner, negating the need
    to bind multiple queues. Stream queue consumers will also be able to read from replicas
    allowing read load to be spread across the cluster.

2. Replay / Time-travelling

    As all current RabbitMQ queue types have destructive consume behaviour, i.e. messages
    are deleted from the queue when a consumer is finished with them, it is not
    possible to re-read messages that have been consumed. Stream queues will allow
    consumers to attach at any point in the log and read from there.

3. Throughput Performance

    No persistent queue types are able to deliver throughput that can compete with
    any of the existing log based messaging systems. Stream queues have been designed
    with performance as a major goal.

4. Large queues

    Most RabbitMQ queues are designed to converge towards the empty state and are
    optimised as such and can perform worse when there are millions of messages on a
    given queue. Stream queues are designed to store larger amounts of data in an
    efficient manner with minimal in-memory overhead.

## <a id="usage" class="anchor" href="#usage">Usage</a>

A client library that can specify [optional queue and consumer arguments](/queues.html#optional-arguments) will
be able to use stream queues.

First we will cover how to declare a stream queue.

### <a id="declaring" class="anchor" href="#declaring">Declaring</a>

To declare a stream queue set the `x-queue-type` queue argument to `stream`
(the default is `classic`). This argument must be provided by a client
at queue declaration time; it cannot be set or changed using a [policy](/parameters.html#policies).
This is because policy definition or applicable policy can be changed dynamically but
queue type cannot. It must be specified at the time of declaration.

Declaring a queue with an `x-queue-type` argument set to `stream` will create a stream queue
with a replica on each configured RabbitMQ node. Stream queues are quorum systems
so uneven cluster sizes is strongly recommended.

After declaration a stream queue can be bound to any exchange just as any other
RabbitMQ queue.

If declaring using [management UI](/management.html), queue type must be specified using
the queue type drop down menu.

Stream queues support 3 additional [queue arguments](/queues.html#optional-arguments)
that are best configured using a [policy](/parameters.html#policies)

 * `max-length-bytes`

Sets the maximum size of the stream in bytes. See retention. Default: not set.

* `max-age`

Sets the maximum age of a stream. See retention. Default: not set.

* `max-segment-size`

Unit: bytes. A stream queue is divided up into fixed size segments on disk.
This setting controls the size of these.
Default: (500000000 bytes). 


### Client Operations


#### Consuming

As stream queues never delete any messages any consumer can start reading/consuming
from any point in the log. This is controlled by the `x-stream-offset` consumer argument.
If it is unspecified the consumer will start reading from the next offset written
to the log after the consumer starts. The following values are supported:

 * "first" - start from the first available message in the log
 * "last" - this starts reading from the last written "chunk" of messages (TODO: explain chunks)
 * "next" - same as not specifying any offset
 * Offset - a numerical value specifying an exact offset to attach to the log at. If
this offset does not exist it will clamp to either the start or end of the log respectively.

#### Other

The following operations can be used in a similar way to classic and quorum queues
but some have some queue specific behaviour.

 * [Declaration](#declaring) (covered above)
 * Queue deletion
 * [Consumption](/consumers.html) (subscription)
    Consumption requires QoS prefetch to be set. The acks works as a credit mechanism
    to advance the current offset of the consumer.

 * Setting [QoS prefetch](#global-qos) for consumers
 * [Consumer acknowledgements](/confirms.html) (keep [QoS Prefetch Limitations](#global-qos) in mind)
 * Cancellation of consumers

## <a id="feature-comparison" class="anchor" href="#feature-comparison">Feature Comparison with Regular Queues</a>

Stream queues are not really queues in the traditional sense and thus do not
align very closely with AMQP 0.9.1 queue semantics. Many features that other queue types
support are not supported and will never be due to the nature of the queue type.

A client library that can use regular mirrored queues will be able to use stream queues
as long as it uses consumer acknowledgements.

Many features will never be supported by stream queues due their non-destructive
read semantics.

### <a id="feature-matrix" class="anchor" href="#feature-matrix">Feature Matrix</a>

| Feature | Classic | Stream |
| :-------- | :------- | ------ |
| [Non-durable queues](/queues.html) | yes | no |
| [Exclusivity](/queues.html) | yes | no |
| Per message persistence | per message | always |
| Membership changes | automatic | manual  |
| [TTL](/ttl.html) | yes | no (but see [Retention](#retention) |
| [Queue length limits](/maxlength.html) | yes | no (but see [Retention](#retention))|
| [Lazy behaviour](/lazy-queues.html) | yes | inherent |
| [Message priority](/priority.html) | yes | no |
| [Consumer priority](/consumer-priority.html) | yes | no |
| [Dead letter exchanges](/dlx.html) | yes | no |
| Adheres to [policies](/parameters.html#policies) | yes | (see [Retention](#retention)) |
| Reacts to [memory alarms](/alarms.html) | yes | no (uses minimal RAM) |
| Poison message handling | no | no |
| Global [QoS Prefetch](#global-qos) | yes | no |

#### Non-durable Queues

Regular queues can be [non-durable](/queues.html). Stream queues are always durable per their
assumed [use cases](#use-cases).

#### Exclusivity

Regular queues can be [exclusive](/queues.html#exclusive-queues). Stream queues are always durable per their
assumed [use cases](#use-cases). They are not meant to be used as [temporary queues](/queues.html#temporary-queues).


#### Lazy Mode

Stream queues store all data directly on disk, after a message has been written
it does not use any memory until it is read. Stream queues are inherently "lazy".


#### <a id="global-qos" class="anchor" href="#global-qos">Global QoS</a>

Stream queues do not support global [QoS prefetch](/confirms.html#channel-qos-prefetch) where a channel sets a single
prefetch limit for all consumers using that channel. If an attempt
is made to consume from a stream queue from a channel with global QoS enabled
a channel error will be returned.

Use [per-consumer QoS prefetch](/consumer-prefetch.html), which is the default in several popular clients.

## <a id="retention" class="anchor" href="#retention">Data Retention</a>

Stream queues are implemented as an immutable append-only disk log. This means that
the log will grow indefinitely until the disk runs out. To avoid this undesirable
scenario it is possible to set a retention configuration per queue which will 
discard the oldest data in the log based on total log data size and/or age.

There are two parameters that control the retention of a stream. These can be combined.
These are either set at declaration time using a queue argument or as a policy which
can be dynamically updated.

 * `max-age`: 

    valid units: Y, M, D, h, m, s

    e.g. `7D` for a week

 * `max-length-bytes`:

    the max total size in bytes

NB: retention is evaluated on per segment basis so there is one more parameter
that comes into effect and that is the segment size of the stream. The stream will
always leave at least one segment in place as long as the segment contains at least
one message.


## <a id="performance" class="anchor" href="#performance">Performance Characteristics</a>

As stream queues persist all data to disks before doing anything it is recommended
to use the fastest disks possible.

Due to the disk I/O-heavy nature of stream queues, their throughput decreases
as message sizes increase.

Just like quorum queues, stream queues are also affected by cluster sizes.
The more replicas a stream queue has, the lower its throughput generally will
be since more work has to be done to replicate data and achieve consensus.




### <a id="replication-factor" class="anchor" href="#replication-factor">Controlling the Initial Replication Factor</a>

TODO

### <a id="replica-management" class="anchor" href="#replica-management">Managing Replicas</a> (Stream Replicas)

Replicas of a stream queue are explicitly managed by the operator. When a new node is added
to the cluster, it will host no stream queue replicas unless the operator explicitly adds it
to a replica set of a stream queue.

When a node has to be decommissioned (permanently removed from the cluster), it must be explicitly
removed from the replica list of all stream queues it currently hosts replicas for.

Two [CLI commands](/cli.html) are provided to perform the above operations,
`rabbitmq-streams add_member` and `rabbitmq-streams delete_member`:

<pre class="lang-bash">
rabbitmq-streams add_replica [-p &lt;vhost&gt;] &lt;queue-name&gt; &lt;node&gt;
</pre>

<pre class="lang-bash">
rabbitmq-streams delete_replica [-p &lt;vhost&gt;] &lt;queue-name&gt; &lt;node&gt;
</pre>

To successfully add and remove replicas the stream coordinator must be
available in the cluster.

Care needs to be taken not to accidentally make a stream unavailable by losing
the quorum whilst performing maintenance operations that involve membership changes.

When replacing a cluster node, it is safer to first add a new node and then decomission the node
it replaces.

### <a id="replica-rebalancing" class="anchor" href="#replica-rebalancing">Rebalancing Replicas</a>

TODO


## <a id="behaviour" class="anchor" href="#behaviour">Behaviour</a>

Every stream queue has a primary writer and zero or more replicas.

A leader is elected when the cluster is first formed and later if the leader
becomes unavailable.

### <a id="leader-election" class="anchor" href="#leader-election">Leader Election and Failure Handling</a>

A stream queue requires a quorum of the declared nodes to be available
to function. When a RabbitMQ node hosting a stream queue's
*leader* fails or is stopped another node hosting one of that
stream queue's *replica* will be elected leader and resume
operations.

Failed and rejoining replicas will re-synchronise ("catch up") with the leader.
In contrast to classic mirrored queues, a temporary replica failure
does not require a full re-synchronization from the currently elected leader. Only the delta
will be transferred if a re-joining replica is behind the leader. This "catching up" process
does not affect leader availability.

Replicas must be explicitly added to
When a new replica is [added](#replica-management), it will synchronise the entire queue state
from the leader, similarly to classic mirrored queues.

### <a id="quorum-requirements" class="anchor" href="#quorum-requirements">Fault Tolerance and Minimum Number of Replicas Online</a>

Consensus systems can provide certain guarantees with regard to data safety.
These guarantees do mean that certain conditions need to be met before they
become relevant such as requiring a minimum of three cluster nodes to provide
fault tolerance and requiring more than half of members to be available to
work at all.

Failure tolerance characteristics of clusters of various size can be described
in a table:

| Cluster node count | Tolerated number of node failures | Tolerant to a network partition |
| :------------------: | :-----------------------------: | :-----------------------------: |
| 1                    | 0                               | not applicable |
| 2                    | 0                               | no |
| 3                    | 1                               | yes |
| 4                    | 1                               | yes if a majority exists on one side |
| 5                    | 2                               | yes |
| 6                    | 2                               | yes if a majority exists on one side |
| 7                    | 3                               | yes |
| 8                    | 3                               | yes if a majority exists on one side |
| 9                    | 4                               | yes |


### <a id="data-safety" class="anchor" href="#data-safety">Data Safety</a>

Stream queues replicate data across multiple nodes and publisher confirms are only
issue once the data has been replicated to a quorum of stream replicas.

Stream queues always store data on disk, however, they do not explicitly flush (fsync)
the data from the operating system page cache to the underlying storage
medium, instead it relies on the operating system to do as and when required.
This means that an uncontrolled shutdown of a server could result in data loss
for replicas hosted on that node. Although theoretically this opens up the possibility
of confirmed data loss, the chances of this happening during normal operation is
very small and the loss of data on a single node would typically just be re-replicated
from the other nodes in the system.

If more data safety is required then consider using quorum queues instead as no
publisher confirms are issued until at least a quorum of nodes have both written _and_
flushed the data to disk.

*_No guarantees are provided for messages that have not been confirmed using
the publisher confirm mechanism_*. Such messages could be lost "mid-way", in an operating
system buffer or otherwise fail to reach the queue leader.


### <a id="availability" class="anchor" href="#availability">Availability</a>

A stream queue should be able to tolerate a minority of queue replicas becoming unavailable
with no or little affect on availability.

Note that depending on the [partition handling strategy](/partitions.html)
used RabbitMQ may restart itself during recovery and reset the node but as long as that
does not happen, this availability guarantee should hold true.

For example, a stream with three replicas can tolerate one node failure without losing availability.
A queue with five replicas can tolerate two, and so on.

If a quorum of nodes cannot be recovered (say if 2 out of 3 RabbitMQ nodes are
permanently lost) the queue is permanently unavailable and
will most likely need operator involvement to be recovered.


## <a id="configuration" class="anchor" href="#configuration">Configuration</a>


## <a id="resource-use" class="anchor" href="#resource-use">Resource Use</a>

Stream queues are typically more light-weight than mirrored and quorum queues.

All data is stored on disk with only unwritten data stored in memory.