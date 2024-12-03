---
title: WAKU-RENDEZVOUS
name: Waku Rendezvous discovery
editor: Simon-Pierre Vivier <simvivier@status.im>
contributors:
---

## Abstract
This document describes the goal,
strategy and usage of the libp2p rendezvous protocol by Waku.

Rendezvous is one of the discovery methods that can be used by Waku.
It supplements Discovery v5 and Waku peer exchange.

## Background and Rationale
How do new nodes join the network is the question that discovery answers.
Discovery must be fast, pertinent and resilient to attacks.
Rendezvous is both fast and allow the discovery of relevant peers,
although it design can be easily abused
due to it's lack of protection against denial of service atacks.
The properties of rendezvous complements well the slower but safer methods like Discv5.
To contribute well, a Waku node must know a sufficient number of peers with
a wide variety of capabilities.
By using rendezvous in combination with
[Discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md#node-discovery-protocol-v5) and
[Waku peer exchange](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md#abstract),
Waku nodes will reach a good number of meaningful peers
faster than through a single discovery method.

## Semantics
Waku rendezvous fully inherits the [libp2p rendezvous semantics](https://github.com/libp2p/specs/blob/master/rendezvous/README.md#rendezvous-protocol).

## Specifications
The namespaces used to register and request MUST be in the format `rs/cluster-id/shard`.
Refer to [RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md) for cluster and shard information.

Every [Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) node SHOULD be initialized as a rendezvous point.

Each relay node that participates in discovery
MUST register with random rendezvous points at regular intervals.

We RECOMMEND a registration interval of  10 seconds.
The node SHOULD register once for every shard it supports,
registering only the namespace corresponding to that shard.

We RECOMMEND that rendezvous points expire registrations after 1 minute,
in order to keep discovered peer records to those recentrly online.

At startup, every Waku node SHOULD discover peers by
sending requests to random rendezvous points,
once for each shard it supports.

We RECOMMEND a maximum of 12 peers will be requested each time.
This number is enough for good GossipSub connectivity and
minimize the load on rendezvous points.

We RECOMMEND that bootstrap nodes participate in rendezvous discovery and
that other discovery methods are used in conjunction and 
continue discovering peers for the lifetime of the local node.

## Future Work

Namespaces will not contain capabilities yet but may in the future. If the need arise nodes could use rendezvous to discover peers with specific capabilities.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References
 - [Discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md#node-discovery-protocol-v5)
 - [Peer Exchange](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md#abstract)
 - [Libp2p Rendezvous](https://github.com/libp2p/specs/blob/master/rendezvous/README.md#rendezvous-protocol)
 - [Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
