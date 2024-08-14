---
title: WAKU-RENDEZVOUS
name: Waku rendez vous discovery
editor: Simon-Pierre Vivier <simvivier@status.im>
contributors:
---

## Abstract
This document describe the goal,
strategy and usage of the libp2p rendezvous protocol by Waku.

Rendezvous is one of the discovery methods that can be used by Waku,
it supplements Discovery v5 and Waku peer exchange but fill a specific role.

## Background
How do new nodes join the network is the question that discovery answers.
Discovery must be fast, pertinent and resilient to attacks.
Rendezvous is both fast and allow the discovery of relevant peers,
although by it's design can be easily abused.
The properties of rendezvous complements well the slower but safer methods like Discv5.
To contribute well, a Waku nodes must know a sufficient number of peers with
a wide variety of capabilities.
By using rendezvous in combination with
[Discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md#node-discovery-protocol-v5) and
[Waku peer exchange](https://github.com/waku-org/specs/blob/master/standards/core/peer-exchange.md#abstract),
Waku nodes will reach a good number of meaningful peers
faster than through a single discovery method.

## Terminology
Please refers to the [libp2p spec](https://github.com/libp2p/specs/blob/master/rendezvous/README.md#rendezvous-protocol).

## Theory
The namespaces used to register and request will be in the format 'rs/cluster-id/shard'.
A registration or request consist of all the shards a nodes supports and their corresponding namespaces.

Every [Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) node will be initialized also as a rendezvous point.
Each relay node will register with a random rendezvous point every 10 seconds.
Each registration will expire after 1 minute.
Each node will roughly have 6 records of itself in the network at any given time.

At startup, every nodes will discover peers by sending a request to a random rendezvous node.
A maximum of 12 peers will be requested.
This number is enough for good GossipSub connectivity and
minimize the load on rendezvous points.
It is assumed that other discovery methods are used in conjunction and
will continue discovering peers for the lifetime of the node.

## Future Work

Namespaces will not contain capabilities yet but may in the future. If the need arise nodes could use rendezvous to discover peers with specific capabilities.
