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

Waku Sync consists of 2 libp2p protocols; reconciliation and transfer.
The Reconciliation protocol finds differences in sets of messages.
The Transfer protocol is used to exchange the differences found with other peers.
The end goal being that peers have the same set of messages.

#### Terminology
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## Reconciliation

**Libp2p Protocol identifier**: `/vac/waku/reconciliation/1.0.0`

The protocol finds differences between 2 peers by comparing _fingerprints_ of _ranges_ of messages.
When the _fingerprints_ are different, _ranges_ are splitted into smaller ones.
This process repeats until _ranges_ include a small number of messages.
At this point, messages are individually compared.

#### Message Ids
Message _Ids_ MUST be composed of the timestamp and the hash of the Waku messages.

The timestamp MUST be the time of creation and
the hash MUST follow the
[deterministic message hashing specification](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing)

> This way the message Ids can always be totally ordered,
first chronologically according to the timestamp and then
disambiguate based on the hash lexical order
in cases where the timestamp is the same.

#### Range Bounds
A _range_ MUST consists of 2 _Ids_, the first bound is
inclusive the second bound exclusive.
The first bound MUST be strictly smaller than the second one.

#### Range Fingerprinting
The _fingerprint_ of a range MUST be the XOR operation applied to
the hash of all message _IDs_ included in that _range_.

#### Range Type
Every _range_ MUST have one of the following types; _fingerprint_, _skip_ or _item set_.

- _Fingerprint_ type contains a _fingerprint_.
- _Skip_ type contains nothing and is used to signal already processed _ranges_.
- _Item set_ type contains message _Ids_ and a _resolved_ boolean.
> _Item sets_ are an optimization, stopping the recursion early can
save network roundtrips.

#### Range Processing
_Ranges_ have to be processed differently according to their types and sent back.

- _Skip_ ranges MUST be merged with other consecutive _skip ranges_.
- **Equal** _fingerprint ranges_ MUST become _skip ranges_.
- **Unequal** _fingerprint ranges_ MUST be splitted into smaller ranges. The new type MAY be either _fingerprint_ or _item set_.
- **Unresolved** _item set_ ranges MUST be checked for differences and marked resolved.
- **Resolved** _item set_ ranges MUST be checked for differences and become skip ranges.

### Delta Encoding
Payloads MUST be delta encoded as follows for efficient transmission of _IDs_ and _ranges_.

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
> The & denote concatenation

1. _varint_ bytes of the delta encoded timestamp &
2. if timestamp is zero, 1 byte for the hash bytes length & the hash bytes &
3. 1 byte, the _range_ type &
4. either
    - 32 bytes _fingerprint_ &
    - _varint_ bytes of the item set length & bytes of every items &
    - if _skip range_, nothing

5. repeat 1 to 4 for all ranges

## Transfer Protocol

**Libp2p Protocol identifier**: `/vac/waku/transfer/1.0.0`

The transfer protocol SHOULD send messages as soon as
a difference is found via reconciliation.
It MUST only accept messages from peers the node is reconciliating with.
New message Ids MUST be added to the reconciliation protocol.
The payload sent MUST follow the wire specification below.

### Wire specification
```protobuf
syntax = "proto3";

package waku.sync.v2;

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

#### Parameters 
Two useful parameters to add to your implementation are partitioning count and the item set threshold.

The partitioning count is the number of time a range is splitted.
Higher value reduce round trips at the cost of computing more fingerprints.

The threshold for which item sets are sent instead of fingerprints.
Higher value sends more items which means higher chance of duplicates but
reduce the amount of round trips overall.

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