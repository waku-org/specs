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

Refer to [RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md) for cluster information.

### Registration and Discovery

Every [Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) node SHOULD be initialized as a rendezvous point.

Each relay node that participates in discovery
MUST register with random rendezvous points at regular intervals.

All relay nodes participating in rendezvous discovery SHOULD advertise their information using `WakuPeerRecord`. For nodes supporting the Mix protocol, the `mix_public_key` field MUST be included. For non-Mix relay nodes, the `mix_public_key` field SHOULD be omitted. The standard libp2p PeerRecord is not used for Waku rendezvous; all advertised records MUST conform to the `WakuPeerRecord` definition.

The RECOMMENDED registration interval is 10 seconds.

It is RECOMMENDED that rendezvous points expire registrations after 1 minute (60 seconds TTL)
to keep discovered peer records limited to those recently online.

At startup, every Waku node supporting Mix SHOULD discover peers by
sending requests to random rendezvous points for the Mix capability namespace.

It is RECOMMENDED a maximum of 12 peers be requested each time.
This number is sufficient for good GossipSub connectivity and
minimizes the load on rendezvous points.

### Peer Records

Nodes advertise their information through `WakuPeerRecord`, a custom peer record structure designed for Waku rendezvous.
Since this is a customPeerRecord, it uses a private multicodec value of `0x300000` as per [multicodec table](https://github.com/multiformats/multicodec/blob/master/table.csv).
The `WakuPeerRecord` is defined as follows:

**WakuPeerRecord fields:**

- `peer_id`: The libp2p PeerId of the node.
- `seqNo`: The time at which the record was created or last updated (Unix epoch, seconds).
- `multiaddrs`: A list of multiaddresses for connectivity.
- `mix_public_key`: The Mix protocol public key (only present for nodes supporting Mix).

**Encoding:**
WakuPeerRecord is encoded as a protobuf message. The exact schema is:

```protobuf
message WakuPeerRecord {
	string peer_id = 1;
	uint64 seqNo = 2;
	repeated string multiaddrs = 3;
	optional bytes mix_public_key = 4;
}
```

When a node discovers peers through rendezvous, it receives the complete `WakuPeerRecord` for each peer, allowing it to make informed decisions about which peers to connect to based on their advertised information.

### Operational Recommendations

It is RECOMMENDED that bootstrap nodes participate in rendezvous discovery and
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
