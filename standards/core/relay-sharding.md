---
title: RELAY-SHARDING
name: Waku v2 Relay Sharding
status: raw
category: Standards Track
tags: [waku/core]
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
  - Simon-Pierre Vivier <simvivier@status.im>
---

## Abstract

This document describes ways of sharding the [Waku relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) topic,
allowing Waku networks to scale in the number of content topics.

> _Note_: Scaling in the size of a single content topic is out of scope for this document.

## Background and Motivation

[Unstructured P2P networks](https://en.wikipedia.org/wiki/Peer-to-peer#Unstructured_networks)
are more robust and resilient against DoS attacks compared to
[structured P2P networks](https://en.wikipedia.org/wiki/Peer-to-peer#Structured_networks)).
However, they do not scale to large traffic loads.
A single [libp2p gossipsub mesh](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#gossipsub-the-gossiping-mesh-router),
which carries messages associated with a single pubsub topic, can be seen as a separate unstructured P2P network
(control messages go beyond these boundaries, but at its core, it is a separate P2P network).
With this, the number of [Waku relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) content topics that can be carried over a pubsub topic is limited.
This prevents app protocols that aim to span many multicast groups (realized by content topics) from scaling.

This document specifies three pubsub topic sharding methods (with varying degrees of automation),
which allow application protocols to scale in the number of content topics.
This document also covers discovery of topic shards.

## Named Sharding

> **_Note:_** As of 2025-07-24 freely named sharding has been deprecated from all Waku implementations.
It is still described here as background for static and automatic sharding.

_Named sharding_ offers apps to freely choose pubsub topic names.
It is RECOMMENDED for App protocols to follow the naming structure detailed in [23/WAKU2-TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md).
With named sharding, managing discovery falls into the responsibility of apps.
For this reason it is NOT RECOMMENDED that app protocols and Waku implementations make use of named sharding.

From an app protocol point of view, a subscription to a content topic `waku2/xxx` on a shard named /mesh/v1.1.1/xxx would look like:

`subscribe("/waku2/xxx", "/mesh/v1.1.1/xxx")`

## Static Sharding

_Static sharding_ is an extension of named sharding that offers a set of shards with _fixed_ names.
Assigning content topics to specific shards is up to app protocols,
but the discovery of these shards is managed by Waku.
This is the RECOMMENDED default format for shards chosen by an app protocol.

Static shards are managed in shard clusters of 1024 shards per cluster.
Waku static sharding can manage $2^{16}$ shard clusters.
Each shard cluster is identified by its index (between $0$ and $2^{16}-1$).

It is RECOMMENDED that, for simplification of configuration and various APIs,
all app-level protocols only interact with `cluster` and `shard`
and never the fully-formed pubsub topic,
which is a concern internal to the Waku implementation.

A specific shard cluster is either globally available to all apps,
specific for an app protocol,
or reserved for automatic sharding (see next section).

> _Note:_ This leads to $2^{16} * 1024 = 2^{26}$ shards for which Waku manages discovery.

App protocols can either choose to use global shards, or app specific shards.

Like the [IANA ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml),
shard clusters are divided into ranges:

| index (range) | usage                |
| ------------- | -------------------- |
| 0 - 15        | reserved             |
| 16 - 65535    | app-defined networks |

The informational RFC [WAKU2-RELAY-STATIC-SHARD-ALLOC](../../informational/relay-static-shard-alloc.md) lists the current index allocations.

The global shard with index 0 and the "all app protocols" range are treated in the same way,
but choosing shards in the global cluster has a higher probability of sharing the shard with other apps.
This offers k-anonymity and better connectivity, but comes at a higher bandwidth cost.

Since the introduction of [the Waku Network](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/64/network.md),
it is RECOMMENDED that apps choose a cluster + shard within the defined range for that network.

The name of the pubsub topic corresponding to a given static shard is specified as

`/waku/2/rs/<cluster_id>/<shard_number>`,

an example for the 2nd shard in the global shard cluster:

`/waku/2/rs/0/2`.

> _Note_: Because _all_ shards distribute payload defined in [14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md) via [protocol buffers](https://developers.google.com/protocol-buffers/),
> the pubsub topic name does not explicitly add `/proto` to indicate protocol buffer encoding.
> We use `rs` to indicate these are _relay shard_ clusters; further shard types might follow in the future.

From an app point of view, a subscription to a content topic `waku2/xxx` on a static shard would look like:

`subscribe("/waku2/xxx", 16, 43)`

for shard 43 of the Status app (which has allocated index 16).

### Discovery

Waku v2 supports the discovery of peers within static shards,
so app protocols do not have to implement their own discovery method.

Nodes add information about their shard participation in their [WAKU2-ENR](./enr.md).
Having a static shard participation indication as part of the ENR allows nodes
to discover peers that are part of shards via [33/WAKU2-DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md) as well as via DNS.

> _Note:_ In the current version of this document,
> sharding information is directly added to the ENR.
> (see Ethereum ENR sharding bit vector [here](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/p2p-interface.md#metadata)
> Static relay sharding supports 1024 shards per cluster, leading to a flag field of 128 bytes.
> This already takes half (including index and key) of the ENR space of 300 bytes.
> For this reason, the current specification only supports a single shard cluster per node.
> In future versions, we will add further (hierarchical) discovery methods.
> We will update [WAKU2-ENR](./enr.md) accordingly, once this RFC moves forward.

This document specifies two ways of indicating shard cluster participation.
The index list SHOULD be used for nodes that participante in fewer than 64 shards,
the bit vector representation SHOULD be used for nodes participating in 64 or more shards.
Nodes MUST NOT use both index list (`rs`) and bit vector (`rsv`) in a single ENR.
ENRs with both `rs` and `rsv` keys SHOULD be ignored.
Nodes MAY interpret `rs` in such ENRs, but MUST ignore `rsv`.

#### Index List

| key  | value                                                                                                                  |
| ---- | ---------------------------------------------------------------------------------------------------------------------- |
| `rs` | <2-byte shard cluster index> &#124; <1-byte length> &#124; <2-byte shard index> &#124; ... &#124; <2-byte shard index> |

The ENR key is `rs`.
The value is comprised of

- a two-byte shard cluster index in network byte order, concatenated with
- a one-byte length field holding the number of shards in the given shard cluster, concatenated with
- two-byte shard indices in network byte order

Example:

| key  | value                                                   |
| ---- | ------------------------------------------------------- |
| `rs` | 16u16 &#124; 3u8 &#124; 13u16 &#124; 14u16 &#124; 45u16 |

This example node is part of shards `13`, `14`, and `45` in the Status main-net shard cluster (index 16).

#### Bit Vector

| key   | value                                                     |
| ----- | --------------------------------------------------------- |
| `rsv` | <2-byte shard cluster index> &#124; <128-byte flag field> |

The ENR key is `rsv`.
The value is comprised of a two-byte shard cluster index in network byte order concatenated with a 128-byte wide bit vector.
The bit vector indicates which shards of the respective shard cluster the node is part of.
The right-most bit in the bit vector represents shard `0`, the left-most bit represents shard `1023`.
The representation in the ENR is inspired by [Ethereum shard ENRs](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#sync-committee-subnet-stability)),
and [this](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#sync-committee-subnet-stability)).

Example:

| key   | value                                  |
| ----- | -------------------------------------- |
| `rsv` | 16u16 &#124; `0x[...]0000100000003000` |

The `[...]` in the example indicates 120 `0` bytes.
This example node is part of shards `13`, `14`, and `45` in the Status main-net shard cluster (index 16).
(This is just for illustration purposes, a node that is only part of three shards should use the index list method specified above.)

## Automatic Sharding

Autosharding is an extension of static sharding, 
where shards can automatically be selected based on content topic.
This is only possible in cases where there exists a clear definition of a "network" combining multiple shards
with a predefined cluster ID and number of shards in the network (`number_of_shards_in_network`).
An example is [the Waku Network](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/64/network.md).
To use autosharding,
the shards corresponding to the network definition MUST be numbered from `0` and monotonously increase to `number_of_shards_in_network - 1`.
The Waku Network SHOULD be considered the default destination for all app-level protocols.
In cases where the Waku Network is not used but autosharding is desired,
the default number of shards in a network SHOULD be considered `1`.

When using autosharding,
shards (pubsub topics) MUST be computed from content topics with the procedure below.

#### Algorithm

Hash using Sha2-256 the concatenation of
the content topic `application` field (UTF-8 string of N bytes) and
the `version` (UTF-8 string of N bytes).
The shard to use is the modulo of the hash by the number of shards in the network.

#### Example

| Field            | Value   | Hex          |
| ---------------- | ------- | ------------ |
| `application`    | "myapp" | 0x6d79617070 |
| `version`        | "1"     | 0x31         |
| `network shards` | 8       | 0x8          |

- SHA2-256 of `0x6d7961707031` is `0x8e541178adbd8126068c47be6a221d77d64837221893a8e4e53139fb802d4928`
- `0x8e541178adbd8126068c47be6a221d77d64837221893a8e4e53139fb802d4928` MOD `8` equals `0`
- The shard to use has index 0

### Content Topics Format for Autosharding

Content topics MUST follow the format in [23/WAKU2-TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md/#content-topic-format).
In addition, a generation prefix MAY be added to content topics.
When omitted default values are used.
Generation default value is `0`.

- The full length format is `/{generation}/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}`
- The short length format is `/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}`

#### Example

- Full length `/0/myapp/1/mytopic/cbor`
- Short length `/myapp/1/mytopic/cbor`

#### Generation and scaling

The generation number monotonously increases and indirectly refers to the total number of shards of a defined network.
In order to scale,
each subsequent generation of a defined network can define a larger `number_of_shards_in_network`,
with the content topics only sharded to the number of shards defined for the corresponding generation of the network.
Generational autosharding MUST be clearly defined for each generation of a network.

For example, consider a specific network defined as the 2 shards `0` and `1` in cluster `32`.
This will automatically be assumed to be generation `0` of this network.
All content topics in the format `/myapp/1/mytopic/cbor` defined for apps on this network will be autosharded to 1 of these 2 shards.
If in future the specifiers of this network want to scale the network to 4 shards (i.e. shards `0` to `3` on cluster `32`),
they MUST define a generation `1` version of the network with `number_of_shards_in_network = 4`.
New content topics for apps on generation `1` of this network MUST be prefixed in the format `/1/myapp/1/mytopic/cbor` to be autosharded into all 4 shards.
Legacy generation `0` content topics will still only be autosharded into the original 2 shards.

<!-- Create a new RFC for each generation spec. -->

#### Topic Design

Content topics have 2 purposes: filtering and routing.
Filtering is done by changing the `{content-topic-name}` field.
As this part is not hashed, it will not affect routing (shard selection).
The `{application-name}` and `{version-of-the-application}` fields do affect routing.
Using multiple content topics with different `{application-name}` field has advantages and disadvantages.
It increases the traffic a relay node is subjected to when subscribed to all topics.
It also allows relay and light nodes to subscribe to a subset of all topics.

### Problems

#### Hot Spots

Hot spots occur (similar to DHTs), when a specific mesh network (shard) becomes responsible for (several) large multicast groups (content topics).
The opposite problem occurs when a mesh only carries multicast groups with very few participants: this might cause bad connectivity within the mesh.

The current autosharding method does not solve this problem.

> _Note:_ Automatic sharding based on network traffic measurements to avoid hot spots in not part of this specification.

#### Discovery

For the discovery of automatic shards this document specifies two methods (the second method will be detailed in a future version of this document).

The first method uses the discovery introduced above in the context of static shards.

The second discovery method will be a successor to the first method,
but is planned to preserve the index range allocation.
Instead of adding the data to the ENR, it will treat each array index as a capability,
which can be hierarchical, having each shard in the indexed shard cluster as a sub-capability.
When scaling to a very large number of shards, this will avoid blowing up the ENR size, and allows efficient discovery.
We currently use [33/WAKU2-DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md) for discovery,
which is based on Ethereum's [discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md).
While this allows to sample nodes from a distributed set of nodes efficiently and offers good resilience,
it does not allow to efficiently discover nodes with specific capabilities within this node set.
Our [research log post](https://vac.dev/wakuv2-apd) explains this in more detail.
Adding efficient (but still preserving resilience) capability discovery to discv5 is ongoing research.
[A paper on this](https://github.com/harnen/service-discovery-paper) has been completed,
but the [Ethereum discv5 specification](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md)
has yet to be updated.
When the new capability discovery is available,
this document will be updated with a specification of the second discovery method.
The transition to the second method will be seamless and fully backwards compatible because nodes can still advertise and discover shard memberships in ENRs.

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](../../informational/adversarial-models.md), especially the parts on k-anonymity.
We will add more on security considerations in future versions of this document.

### Receiver Anonymity

The strength of receiver anonymity, i.e. topic receiver unlinkablity,
depends on the number of content topics (`k`), as a proxy for the number of peers and messages, that get mapped onto a single pubsub topic (shard).
For _named_ and _static_ sharding this responsibility is at the app protocol layer.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [Unstructured P2P network](https://en.wikipedia.org/wiki/Peer-to-peer#Unstructured_networks)
- [33/WAKU2-DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md)
- [WAKU2-ENR](./enr.md)
- [23/WAKU2-TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md)
- [Ethereum ENR sharding bit vector](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/p2p-interface.md#metadata)
- [Ethereum discv5 specification](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md)
- [Research log: Waku Discovery](https://vac.dev/wakuv2-apd)
- [WAKU2-RELAY-STATIC-SHARD-ALLOC](../../informational/relay-static-shard-alloc.md)
- [14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md)
- [64/WAKU2-NETWORK](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/64/network.md)
