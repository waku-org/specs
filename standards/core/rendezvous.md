---
title: WAKU-RENDEZVOUS
name: Waku Rendezvous discovery
editor: Prem Chaitanya Prathi <premprathi@proton.me>
contributors: Simon-Pierre Vivier <simvivier@status.im>
---

## Abstract

This document describes the goal,
strategy, usage, and changes to the libp2p rendezvous protocol by Waku.

Rendezvous is one of the discovery methods that can be used by Waku.
It supplements Discovery v5 and Waku peer exchange.

## Background and Rationale

Waku nodes need a discovery mechanism that is both rapid and robust against attacks. Rendezvous enables quick identification of relevant peers, complementing slower but more resilient approaches like Discv5. For healthy network participation, nodes must connect to a diverse set of peers. However, the rendezvous protocol is vulnerable to denial of service attacks and depends on centralized rendezvous points, so it is best used in conjunction with other discovery methods such as Discv5.

Ethereum Node Record (ENR) size constraints make it impractical to encode additional metadata, such as Mix public keys, in ENRs for Discv5-based discovery. Rendezvous, using a custom peer record (`WakuPeerRecord`), allows nodes to advertise Mix-specific information—including the Mix public key—without being restricted by ENR size. This serves as an interim solution until a comprehensive capability discovery mechanism is available.

By combining rendezvous with
[Discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md#node-discovery-protocol-v5) and
[34/WAKU2-PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md#abstract),
Waku nodes can more quickly reach a meaningful set of peers
than by relying on a single discovery method.

## Semantics

Waku rendezvous extends the [libp2p rendezvous semantics](https://github.com/libp2p/specs/blob/master/rendezvous/README.md#rendezvous-protocol) by using `WakuPeerRecord` instead of the standard libp2p `PeerRecord`.
This allows nodes to advertise additional Waku-specific metadata beyond what is available in the standard libp2p peer record.

## Specifications

**Libp2p Protocol identifier**: `/vac/waku/rendezvous/1.0.0`

### Namespace Format

The namespaces used to register and request MUST be in the format `rs/<cluster-id>/<capability>`.

This allows for discovering peers with specific capabilities within a given cluster.
Currently, this is used for Mix protocol discovery where the capability field specifies `mix`.

Refer to [RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md) for cluster and shard information.

### Registration and Discovery

Every [Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) node SHOULD be initialized as a rendezvous point.

Each relay node that participates in discovery
MUST register with random rendezvous points at regular intervals.
Nodes supporting the Mix protocol MUST advertise their `WakuPeerRecord`, which includes their multiaddresses, supported protocols, and Mix public key.

We RECOMMEND a registration interval of 10 seconds.

We RECOMMEND that rendezvous points expire registrations after 1 minute (60 seconds TTL)
to keep discovered peer records limited to those recently online.

At startup, every Waku node supporting Mix SHOULD discover peers by
sending requests to random rendezvous points for the Mix capability namespace.

We RECOMMEND a maximum of 12 peers be requested each time.
This number is sufficient for good GossipSub connectivity and
minimizes the load on rendezvous points.

### Peer Records

Nodes advertise their information through `WakuPeerRecord`, which includes:

- Multiaddresses for connectivity
- Supported protocol codecs
- Mix protocol public key (for nodes supporting the Mix protocol)

When a node discovers peers through rendezvous, it receives the complete `WakuPeerRecord` for each peer,
allowing it to make informed decisions about which peers to connect to based on their advertised information.

### Operational Recommendations

We RECOMMEND that bootstrap nodes participate in rendezvous discovery and
that other discovery methods are used in conjunction and
continue discovering peers for the lifetime of the local node.

For resource-constrained devices or light clients, a client-only mode MAY be used
where nodes only query for peers without acting as rendezvous points themselves
and without advertising their own peer records.

## Future Work

The protocol currently supports advertising Mix-specific capabilities (Mix public keys) through `WakuPeerRecord`.
Future enhancements could include:

- Extending `WakuPeerRecord` to advertise other Waku protocol capabilities (Relay, Store, Filter, Lightpush, etc.)
- Supporting shard-based namespaces (e.g., `rs/<cluster-id>/<shard>`) for general relay peer discovery without capability filtering
- Batch registration support allowing nodes to register across multiple namespaces in a single request

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

- [Discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md#node-discovery-protocol-v5)
- [ENR](https://github.com/ethereum/devp2p/blob/master/enr.md)
- [34/WAKU2-PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md#abstract)
- [Libp2p Rendezvous](https://github.com/libp2p/specs/blob/master/rendezvous/README.md#rendezvous-protocol)
- [Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
