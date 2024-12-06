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
which enables the syncronization of messages between 2 Store nodes.

# Specification

Waku Sync consists of 2 protocols; reconciliation and transfer.
Reconciliation is the process of finding differences in 2 sets of message hashes.
Transfer is then used to bilateraly send messages to the other peer.
The end goal being that both peers have the same set of hashes and messages.

#### Terminology
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## Reconciliation

**Libp2p Protocol identifier**: `/vac/waku/reconciliation/1.0.0`

#### Message Ids
Message Ids MUST be composed of the timestamp and the hash of the Waku messages.

The timestamp MUST be the time of creation and
the hash MUST follow the
[deterministic message hashing specification](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing)

> This way the message Ids can always be totally ordered.
Chronologically according to the timestamp and
disambiguate based on the hash lexical order
in cases where the timestamp is the same.

#### Range Bounds
A range MUST consists of 2 Id bounds, the first bound is
inclusive the second bound exclusive.
The first bound MUST be strictly smaller than the second one.

#### Range Fingerprinting
The fingerprint of a range MUST be the XOR operation applied to
all the hashes of the messages included in that range.

#### Range Type
Every range MUST have one of the following types; skip, fingerprint or item set.

- Skip type is used to signal already processed ranges that MUST be ignored.
- Fingerprint type signify that fingerprints MUST be compared when received.
- Item set type contain multiple message Ids that MUST all be compared when received.
> Item sets are an optimization, stopping the recursion early can
save network roundtrips.

#### Range Processing
Ranges have to be processed differently acording to their types.

- Skip ranges MUST be merged with other consequtive ones if possible.
- Equal fingerprint ranges MUST become skip ranges.
- Unequal fingerprint ranges MUST be splitted into smaller ranges. The new type MAY be either fingerprint or item set.
- Unresolved item set ranges MUST be checked for differences and marked resolved.
- Resolved item set ranges MUST be checked for differences and become skip ranges.

### Delta Encoding
For efficient transmition of timestamps, hashes and ranges. Payloads are delta encoded as follow.

All ranges to be transmitted MUST be ordered and only upper bounds used.
> Inclusive lower bounds can be omitted because they are always
the same as the exclusive upper bounds of the previous range or zero.

To achieve this, it MAY be needed to add skip ranges.
> For example, a skip range can be added with
an exclusive upper bound equal to the first range lower bound.
This way the receiving peer knows to ignore the range from zero to the start of the sync window.

Every timestamps after the first MUST be noted as the difference from the previous one.
If the timestamp is the same, zero MUST be used and the hash MUST be added.
The added hash MUST be trucated up to and including the first differetiating byte.

| Timestamp | Hash | Timestamp (encoded) | Hash (encoded) 
| - | - | - | -
| 1000 | 0x4a8a769a... | 1000 | -
| 1002 | 0x351c5e86... | 2 | -
| 1002 | 0x3560d9c4... | 0 | 0x3560
| 1003 | 0xbeabef25... | 1 | -

#### Varints
TODO

#### Payload encoding
The wire level payload MUST be encoded as follow.
> The & denote concatenation

1. varint bytes of the delta encoded timestamp &
2. if timestamp is zero, delta encoded hash bytes  &
3. 1 byte, the range type &
4. either
    - 32 bytes fingerprint &
    - varint bytes of the item set length & bytes of every items &
    - if skip range, nothing

5. repeat 1 to 4 for all ranges

## Transfer Protocol

**Libp2p Protocol identifier**: `/vac/waku/transfer/1.0.0`

TODO

should not accept messages from peers not being syncing with.

should send message as soon as a diff is found.

in the future verify RLN proof of messages.

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
The flexibitity of the protocol implies that much is left to the implementers.
What will follow is NOT part of the specification.
This section was created to inform implementations.

#### Parameters 
#TODO fix copy pasta from research issue

T -> Item set threshold. If a range length is <= than T, all items are sent. Higher T sends more items which means higher chance of duplicates but reduce the amount of round trips overall.

B -> Partitioning count. When recursively splitting a range, it is split into B sub ranges. Higher B reduce round trips at the cost of computing more fingerprints.

#### Storage
The storage implementation should reflect the context.
Most messages that will be added will be recent and
removed messages will be older ones.
When differences are found some messages will have to be inserted randomly.
It is expected to be a less likely case than time based insertion and removal.
Last but not least it must be optimized for fingerprinting
as it is the most often used operation.

TODO mention trees vs arrays???

#### Sync Window
TODO rephrase

We also offset the sync window by 20 seconds in the past.
The actual start of the sync range is T-01:00:20 and the end T-00:00:20
This is to handle the inherent jitters of GossipSub.
In other words, it is the amount of time needed to confirm if a message is missing or not.

#### Sync Interval
TODO rephrase

Ad-hoc syncing can be useful in some cases but continuous periodic sync
minimize the differences in messages stored across the network.
Syncing early and often is the best strategy.
The default used in nwaku is 5 minutes interval between sync with a range of 1 hour.

#### Peer Choice
TODO rephrase

Peering strategies can lead to inadvertently segregating peers and reduce sampling diversity.
We randomly select peers to sync with for simplicity and robustness.

A good strategy can be devised but we chose not to.

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