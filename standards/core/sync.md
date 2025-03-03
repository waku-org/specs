---
title: WAKU-SYNC
name: Waku Sync
editor: Simon-Pierre Vivier <simvivier@status.im>
contributors:
 - Prem Chaitanya Prathi <prem@status.im>
 - Hanno Cornelius <hanno@status.im>
---

# Abstract
This specification explains `WAKU-SYNC`
which enables the synchronization of messages between nodes storing sets of [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message)

# Specification

Waku Sync consists of two libp2p protocols: `reconciliation` and `transfer`.
The Reconciliation protocol finds differences in sets of messages.
The Transfer protocol is used to exchange the differences found with other peers.
The end goal being that peers have the same set of messages.

#### Terminology
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## Reconciliation

**Libp2p Protocol identifier**: `/vac/waku/reconciliation/1.0.0`

The protocol finds differences between two peers by
comparing _fingerprints_ of _ranges_ of message _IDs_.
_Ranges_ are encoded into payloads,
exchanged between the peers and when
the range _fingerprints_ are different, split into smaller (sub)ranges.
This process repeats until _ranges_ include a small number of messages.
At this point lists of message _IDs_ are sent for comparison
instead of _fingerprints_ over entire ranges of messages.

#### Overview
The `reconciliation` protocol operate on the following heuristic:
1. The requestor chooses a sync range including time, cluster id and topics.
2. The range is encoded into a payload and sent.
3. The requestee receives the payload and decodes it.
4. The range is processed and, if a difference with the local range is detected, a set of subranges are produced.
5. The new ranges are encoded and sent.
6. This process repeats while differences found are sent to the `transfer` protocol.
7. The synchronization ends when all ranges have been processed and no differences are left.

#### Message IDs
Message _IDs_ MUST be composed of the timestamp and the hash of the [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message).

The timestamp MUST be the time of creation and
the hash MUST follow the
[deterministic message hashing specification](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing)

> This way the message IDs can always be totally ordered,
first chronologically according to the timestamp and then
disambiguated based on the hash lexical order
in cases where the timestamp is the same.

#### Range Bounds
A _range_ MUST consist of two _IDs_, the first bound is
inclusive the second bound exclusive.
The first bound MUST be strictly smaller than the second one.

#### Range Fingerprinting
The _fingerprint_ of a range MUST be the XOR operation applied to
the hash of all message _IDs_ included in that _range_.

#### Range Type
Every _range_ MUST have one of the following types; _fingerprint_, _skip_ or _item set_.

- _Fingerprint_ type contains a _fingerprint_.
- _Skip_ type contains nothing and is used to signal already processed _ranges_.
- _Item set_ type contains message _IDs_ and a _resolved_ boolean.
> _Item sets_ are an optimization, sending multiple _IDs_ instead of
recursing further reduce the number of round-trips.

#### Sync Scope
On payload reception, the intersection of the two peers topic sets is used as the sync scope.
If the intersection is empty the sync is aborted.
If a peer does not specify a set of supported topics,
the protocol assumes all topics are supproted for that peer.

#### Range Processing
_Ranges_ have to be processed differently according to their types.

- _Fingerprint_ ranges MUST be compared.
  - **Equal** ranges MUST become _skip ranges_.
  - **Unequal** ranges MUST be split into smaller _fingerprint_ or _item set_ ranges based on a implementation specific threshold.
- **Unresolved** _item set_ ranges MUST be compared, differences sent to the `transfer` protocol and marked resolved.
- **Resolved** _item set_ ranges MUST be compared, differences sent to the `transfer` protocol and become skip ranges.
- _Skip_ ranges MUST be merged with other consecutive _skip ranges_.

In the case where only skip ranges remains, the synchronization is done.

### Delta Encoding
_Ranges_ and timestamps MUST be delta encoded as follows for efficient transmission.

All _ranges_ to be transmitted MUST be ordered and only upper bounds used.
> Inclusive lower bounds can be omitted because they are always
the same as the exclusive upper bounds of the previous range or zero.

To achieve this, it MAY be needed to add _skip ranges_.
> For example, a _skip range_ can be added with
an exclusive upper bound equal to the first range lower bound.
This way the receiving peer knows to ignore the range from zero to the start of the sync time window.

Every _ID_'s timestamps after the first MUST be noted as the difference from the previous one.
If the timestamp is the same, zero MUST be used and the hash MUST be added.
The added hash MUST be truncated up to and including the first differentiating byte.

| Timestamp | Hash | Timestamp (encoded) | Hash (encoded) 
| - | - | - | -
| 1000 | 0x4a8a769a... | 1000 | -
| 1002 | 0x351c5e86... | 2 | -
| 1002 | 0x3560d9c4... | 0 | 0x3560
| 1003 | 0xbeabef25... | 1 | -

#### Varints
All _varints_ MUST be little-endian base 128 variable length integers (LEB128) and minimally encoded.

#### Payload encoding
The wire level payload MUST be encoded as follow.

> Refer to [RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md#static-sharding)
RFC for cluster and shard specification.

> Please note that for each steps, bytes are concatenated.

1. _Varint_ bytes of the node's cluster ID
2. _Varint_ bytes of the number of pubsub topics
3. For each pubsub topic, if any

    a. _varint_ bytes of the pubsub topic length

    b. bytes content of the pubsub topic

4. _Varint_ bytes of the number of content topics
5. For each content topic, if any

    a. _varint_ bytes of the content topic length

    b. bytes content of the content topic

6. For each range
    
    a. _varint_ bytes of the delta encoded timestamp
    
    b. if timestamp is zero
      - 1 byte for the partial hash length
      - the hash bytes content
    
    c. 1 byte, the _range_ type

    d. either
      - 32 bytes _fingerprint_ or
      - _varint_ bytes of the item set length and the bytes of every items or
      - if _skip range_, nothing

## Transfer Protocol

**Libp2p Protocol identifier**: `/vac/waku/transfer/1.0.0`

The transfer protocol SHOULD send messages as soon as
a difference is found via reconciliation.
It MUST only accept messages from peers the node is reconciliating with.
New message IDs MUST be added to the reconciliation protocol.
The payload sent MUST follow the wire specification below.

### Wire specification
```protobuf
syntax = "proto3";

package waku.sync.transfer.v1;

import "waku/message/v1/message.proto";

message WakuMessageAndTopic {
  // Full message content and associated pubsub_topic as value
  optional waku.message.v1.WakuMessage message = 1;
  optional string pubsub_topic = 2;
}
```

# Implementation
The flexibility of the protocol implies that much is left to the implementers.
What will follow is NOT part of the specification.
This section was created to inform implementations.

#### Cluster, Pubsub and Content Topics
To prevent nodes from synchronizing messages from cluster and topics they don't support,
cluster and topics information is added to each payload.

#### Parameters 
Two useful parameters to add to your implementation are partitioning count and the item set threshold.

The partitioning count is the number of time a range is split.
A higher value reduces round trips at the cost of computing more fingerprints.

The item set threshold determines when item sets are sent instead of fingerprints.
A higher value sends more items which means higher chance of duplicates but
reduces the amount of round trips overall.

#### Storage
The storage implementation should reflect the context.
Most messages that will be added will be recent and
removed messages will be older ones.
When differences are found some messages will have to be inserted randomly.
It is expected to be a less likely case than time based insertion and removal.
Last but not least it must be optimized for fingerprinting
as it is the most often used operation.

#### Sync Interval
Ad-hoc syncing can be useful in some cases but continuous periodic sync
minimize the differences in messages stored across the network.
Syncing early and often is the best strategy.
The default used in Nwaku is 5 minutes interval between sync with a range of 1 hour.

#### Sync Window
By default we offset the sync window by 20 seconds in the past.
The actual start of the sync range is T-01:00:20 and the end T-00:00:20 in most cases.
This is to handle the inherent jitters of GossipSub.
In other words, it is the amount of time needed to confirm if a message is missing or not.

#### Peer Choice
Wrong peering strategies can lead to inadvertently segregating peers and
reduce sampling diversity.
Nwaku randomly select peers to sync with for simplicity and robustness.

More sophisticated strategies may be implemented in future.

## Attack Vectors
Nodes using `WAKU-SYNC` are fully trusted.
Message hashes are assumed to be of valid messages received via Waku Relay or Light push.

Further refinements to the protocol are planned
to reduce the trust level required to operate.
Notably by verifying messages RLN proof at reception.

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References
 - [RBSR](https://github.com/AljoschaMeyer/rbsr_short/blob/main/main.pdf)
 - [Negentropy Explainer](https://logperiodic.com/rbsr.html)
 - [Master Thesis](https://github.com/AljoschaMeyer/master_thesis/blob/main/main.pdf)