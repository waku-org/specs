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
which enables the synchronization of messages between nodes storing sets of [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message)s.

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

The `reconciliation` protocol finds the differences between two sets of [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message) on different nodes.
It assumes that each [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message) maps to a uniquely identifying `SyncID`,
which is maintained in an ordered set within each node.
An ordered set of `SyncID`s is termed a `Range`.
This implies that any contiguous subset of a `Range` is also a `Range`.
In other words, the `reconciliation` protocol allows two nodes to find differences between `Range`s of `SyncID`s,
which would map to an equivalent difference in cached [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message)s.
These terms, the wire protocol and message flows are explained below.

### Wire protocol

#### Reconcilation payload

Nodes participating in the `reconciliation` protocol exchange encoded `RangesData` messages.

The `RangesData` structure represents a complete `reconciliation` payload:

| Field         | Type                     | Description                              |
|---------------|--------------------------|------------------------------------------|
| cluster       | varint                   | Cluster ID of the sender                 |
| shards        | seq[varint]              | Shards supported by the sender           |
| ranges        | seq[Range]               | Sequence of ranges       |

The `cluster` and `shards` fields
represent the sharding elements as defined in [RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md#static-sharding).
The `ranges` field contain a sequence of ranges for reconciliation.

We identify the following subtypes:

##### Range

A `Range` represents a representation of the bounds, type and, optionally, an encoded representation of the content of a range.

| Field         | Type                     | Description                              |
|---------------|--------------------------|------------------------------------------|
| bounds        | RangeBounds              | The bounds of the range                 |
| type          | RangeType                | The type of the range           |
| Option(content)| Fingerprint OR ItemSet   | Optional field depending on `type`. If set, possible values are a `Fingerprint` of the range, or the complete `ItemSet` in the range |

##### RangeBounds

`RangeBounds` defines a `Range` by two bounding `SyncID` values, forming a time-hash interval:

| Field   | Type    | Description                          |
|---------|---------|--------------------------------------|
| a       | SyncID  | Lower bound (inclusive)              |
| b       | SyncID  | Upper bound (exclusive)              |

The lower bound MUST be strictly smaller than the upper bound.

##### SyncID

A `SyncID` consists of a message timestamp and hash to uniquely identify a [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message).

| Field     | Type       | Description                               |
|-----------|------------|-------------------------------------------|
| timestamp | varint     | Timestamp of the message (nanoseconds)    |
| hash      | seq[Byte]  | 32-byte message hash                 |

The timestamp MUST be the time of message creation and
the hash MUST follow the [deterministic message hashing specification](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing).
SyncIDs MUST be totally ordered by timestamp first, then by hash to disambiguate messages with identical timestamps.

##### RangeType

A `RangeType` indicates the type of `content` encoding for the `Range`:

| Type        | Value | Description                                     |
|-------------|-------|-------------------------------------------------|
| Skip        | 0     | Range has been processed and `content` field is empty |
| Fingerprint | 1     | Range is encoded in `content` field as a 32-byte `Fingerprint`     |
| ItemSet     | 2     | Range is encoded in `content` field as an `ItemSet` of `SyncID`s   |

##### Fingerprint

A `Fingerprint` is a compact 32-byte representation of all message hashes within a `Range`.
It MUST be implemented as an XOR of all contained hashes,
extracted from the `SyncID`s present in the `Range`.
Fingerprints allow for efficient difference detection without transmitting complete message sets.

##### ItemSet

An `ItemSet` contains a full representation of all `SyncID`s within a `Range`
and a reconciliation status:

| Field      | Type       | Description                          |
|------------|------------|--------------------------------------|
| elements   | seq[`SyncID`] | Sequence of `SyncID`s in the `Range` |
| reconciled | bool       | Whether the `Range` has been reconciled  |

#### Payload encoding

We can now define a heuristic for encoding the `RangesData` payload.
All varints MUST be encoded according to the [specified varint encoding](#varint-encoding) procedure.

Concatenate each of the following, in order:
1. Encode the cluster ID as a varint
2. Encode the number of shards as a varint
3. For each shard, encode the shard as a varint and concatenate it to the previous
4. Encode the `ranges`, according to ["Delta encoding for Ranges"](#delta-encoding-for-ranges).

##### Varint encoding

All variable integers (varints) MUST be encoded as little-endian base 128 variable-length integers (LEB128) and MUST be minimally encoded.

##### Delta encoding for Ranges

The `ranges` field contain a sequence of `Ranges`.

It can be delta encoded as follows:

For each `Range` element, concatenate the following:
  1. From the `RangeBounds`, select only the `SyncID` representing the _upper bound_ of that range.
  Inclusive lower bounds are omitted because they are always the same as the exclusive upper bounds of the previous range.
  The first range is always assumed to have a lower bound `SyncID` of `0` (both the `timestamp` and `hash` are `0`).
  2. [Delta encode](#delta-encoding-for-sequential-syncids) the selected `SyncID` by comparing it to the previous ranges' `SyncID`s.
  The first range's `SyncID` will be fully encoded, as described in that section.
  3. Encode the `RangeType` as a single byte and concatenate it to the delta encoded `SyncID`.
  4. If the `RangeType` is:
     - _Skip_: encode nothing more
     - _Fingerprint_: encode the 32-byte fingerprint
     - _ItemSet_:
       - Encode the number of elements as a varint
       - Delta encode the item set, according to ["Delta encoding for ItemSets"](#delta-encoding-for-itemsets)

##### Delta encoding for sequential SyncIDs

Sequential `SyncID`s can be delta encoded to minimize payload size.

Given an ordered sequence of `SyncID`s:
1. Encode the first timestamp in full
2. Delta encode subsequent timestamps, i.e., encode only the difference from the previous timestamp
3. If the timestamps are identical, encode a `SyncID` `hash` delta as follows:
3.1. Compared to the previous hash, truncate the hash up to and including the first differentiating byte
3.2. Encode the number of bytes in the truncated hash (as a single byte)
3.3. Encode the truncated hash

See the table below as an example:

| Timestamp | Hash | Encoded Timestamp (diff from previous) | Encoded Hash (length + all bytes up to first diff)
| - | - | - | -
| 1000 | 0x4a8a769a... | 1000 | -
| 1002 | 0x351c5e86... | 2 | -
| 1002 | 0x3560d9c4... | 0 | 0x023560
| 1003 | 0xbeabef25... | 1 | -

##### Delta encoding for ItemSets

An `ItemSet` is delta encoded as follows:

Concatenate each of the following, in order:
1. From the first `SyncID`, encode the timestamp in full
2. From the first `SyncID`, encode the hash in full
3. For each subsequent `SyncID`:
   - Delta encode the timestamp, i.e., encode only the difference from the previous `SyncID`'s timestamp
   - Append the encoded hash in full
4. Encode a single byte for the `reconciled` boolean (0 or 1)

### Reconciliation Message Flow

The `reconciliation` message flow is triggered by an `initiator` with an initial `RangesData` payload.
The selection of sync peers
and triggers for initiating a `reconciliation` is out of scope of this document.

The response to a `RangesData` payload is another `RangesData` payload, according to the heuristic below.
The syncing peers SHOULD continue exchanging `RangesData` payloads until all ranges have been processed.

The output of a `reconciliation` flow is a set of differences in `SyncID`s.
These could either be `SyncID`s present in the remote peer and missing locally,
or `SyncID`s present locally but missing in the remote peer.
Each sync peer MAY use the `transfer` protocol to exchange the full messages corresponding to these computed differences,
proactively transferring messages missing in the remote peer.

#### Initial Message

The initiator
1. Selects a time range to sync
2. Computes a fingerprint for the entire range
3. Constructs an initial `RangesData` payload with:
   - The initiator's cluster ID
   - The initiator's supported shards
   - A single range of type `Fingerprint` covering the entire sync period
   - The fingerprint for this range
4. [Delta encodes](#payload-encoding) the payload
5. Sends the payload using libp2p length-prefixed encoding

#### Responding to a `RangesData` payload

Each syncing peer performs the following in response to a `RangesData` payload:

The responder:
1. Receives and decodes the payload
2. If shards and cluster match, processes each range:
   - If the received range is of type `Skip`, ignores it.
   - If the received range is of type `Fingerprint`, computes the fingerprint over the local matching range
      - If the local fingerprint matches the received fingerprint, includes this range in the response as type `Skip`
      - If the local fingerprint does _not_ match the received fingerprint:
         - If the corresponding range is small enough, includes this range in the response as type `ItemSet` with all `SyncID`s
         - If the corresponding range is too large, divide it into subranges.
         For each subrange, if the range is small enough, includes it in the response as type `ItemSet` with all `SyncID`s.
         If the subrange is too large, includes it in the response as type `Fingerprint`.
   - If the received range is of type `ItemSet`, compares it to the local items in the corresponding range
      - If there are any differences between the local and remote items, adds these as part of the output of the `reconciliation` procedure.
      At this point, the syncing peers MAY start to exchange the full messages corresponding to differences using the `transfer` protocol.
      - If the received `ItemSet` is _not_ marked as `reconciled` by the remote peer, includes the corresponding local range in the response as type `ItemSet`.
      - If the received `ItemSet` is marked as `reconciled` by the remote peer, includes the corresponding local range in the response as type `Skip`.
3. If shards or cluster don't match:
   - Responds with an empty payload
4. Delta encodes the response
5. Sends the response using libp2p length-prefixed encoding

This process continues until the syncing peer crafts an empty response,
i.e., when all ranges have been processed and reconciled by both syncing peers and there are no differences left.

## Transfer Protocol

**Libp2p Protocol identifier**: `/vac/waku/transfer/1.0.0`

Once the `reconciliation` protocol starts finding differences in `SyncID`s,
the `transfer` protocol MAY be used to exchange actual message contents between peers.
A node using `transfer` SHOULD proactively send [`14/WAKU2-MESSAGE`](https://rfc.vac.dev/waku/standards/core/14/message)s missing in the remote party.
Nodes SHOULD only accept incoming transfers from peers for which they have an active reconciliation session.
The `SyncID`s corresponding to messages received via `transfer`,
MUST be added to the corresponding `Range` tracked by the `reconciliation` protocol.
The `transfer`payload MUST follow the wire specification below.

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

## Implementation Suggestions

The flexibility of the protocol implies that much is left to the implementers.
What will follow is NOT part of the specification.
This section was created to inform implementations.

### Cluster & Shards

To prevent nodes from synchronizing messages from shard they don't support,
cluster and shards information has been added to each payload.
On reception, if two peers don't share the same set of shards the sync is aborted.

### Parameters 

Two useful parameters to add to your implementation are partitioning count and the item set threshold.

The partitioning count is the number of time a range is split.
A higher value reduces round trips at the cost of computing more fingerprints.

The item set threshold determines when item sets are sent instead of fingerprints.
A higher value sends more items which means higher chance of duplicates but
reduces the amount of round trips overall.

### Storage

The storage implementation should reflect the context.
Most messages that will be added will be recent and
removed messages will be older ones.
When differences are found some messages will have to be inserted randomly.
It is expected to be a less likely case than time based insertion and removal.
Last but not least it must be optimized for fingerprinting
as it is the most often used operation.

### Sync Interval

Ad-hoc syncing can be useful in some cases but continuous periodic sync
minimize the differences in messages stored across the network.
Syncing early and often is the best strategy.
The default used in Nwaku is 5 minutes interval between sync with a range of 1 hour.

### Sync Window

By default we offset the sync window by 20 seconds in the past.
The actual start of the sync range is T-01:00:20 and the end T-00:00:20 in most cases.
This is to handle the inherent jitters of GossipSub.
In other words, it is the amount of time needed to confirm if a message is missing or not.

### Peer Choice

Wrong peering strategies can lead to inadvertently segregating peers and
reduce sampling diversity.
Nwaku randomly select peers to sync with for simplicity and robustness.

More sophisticated strategies may be implemented in future.

## Security/Privacy Considerations

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