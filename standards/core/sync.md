---
title: WAKU-SYNC
name: Waku Sync
editor: Simon-Pierre Vivier <simvivier@status.im>
contributors:
 - Prem Chaitanya Prathi <prem@status.im>
 - Hanno Cornelius <hanno@status.im>
---

# Abstract
This specification explains the `WAKU-SYNC` protocol
which enables the reconciliation of two sets of message hashes
in the context of keeping multiple Store nodes syncronized.
Waku Sync is a wrapper around
[Negentropy](https://github.com/hoytech/negentropy) a [range-based set reconciliation protocol](https://logperiodic.com/rbsr.html).

# Specification

**Protocol identifier**: `/vac/waku/sync/1.0.0`

## Terminology
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

The term Negentropy refers to the protocol of the same name.
Negentropy payload refers to
the messages created by the Negentropy protocol.
Client always refers to the initiator
and the server the receiver of the first payload.

## Design Requirements
Nodes enabling Waku Sync SHOULD
manage and keep message hashes in a local cache
for the range of time
during which syncronization is required.
Nodes SHOULD use the same time range,
for Waku we chose one hour as the global default.
Waku Relay or Light Push protocol MAY be enabled
and used in conjuction with Sync
as a source of new message hashes
for the cache.

Nodes MAY use the Store protocol
to request missing messages once reconciliation is complete
or to provide messages to requesting clients.

## Payload

```protobuf
syntax = "proto3";

package waku.sync.v1;

message SyncPayload {
  optional bytes negentropy = 1;

  repeated bytes hashes = 20;
}
```

## Session Flow
A client initiate a session with a server
by sending a `SyncPayload` with
only the `negentropy` field set to it.
This field MUST contain
the first negentropy payload
created by the client
for this session.

The server receives a `SyncPayload`.
A new negentropy payload is computed from the received one.
The server sends back a `SyncPayload` to the client.

The client receives a `SyncPayload`.
A new negentropy payload OR an empty one is computed.
If a new payload is computed then
the exchanges between client and server continues until
the client computes an empty payload.
This client computation also outputs any hash differences found,
those MUST be stored.
In the case of an empty payload,
the client MUST send back a `SyncPayload`
with all the hashes previoudly found in the `hashes` field and
an empty `nengentropy` field.

# Attack Vectors
Nodes using `WAKU-SYNC` are fully trusted.
Message hashes are assumed to be of valid messages received via Waku Relay or Light push.

Further refinements to the protocol are planned
to reduce the trust level required to operate.
Notably by verifing messages RLN proof at reception.

# Implementation
The following is not part of the specifications but good to know implementation details.

### Interval
Ad-hoc syncing can be useful in some cases but continueous periodic sync
minimise the differences in messages stored across the network.
Syncing early and often is the best strategy.
The default used in nwaku is 5 minutes interval between sync with a range of 1 hour.

### Range
We also offset the sync range by 20 seconds in the past.
The actual start of the sync range is T-01:00:20 and the end T-00:00:20
This is to handle the inherent jitter of GossipSub.
In other words, it is the amount of time needed to confirm if a message is missing or not.

### Storage
The storage implementation should reflect the Waku context.
Most messages that will be added will be recent and
all removed messages will be older ones.
When differences are found some messages will have to be inserted randomly.
It is expected to be a less likely case than time based insertion and removal.
Last but not least it must be optimized for sequential read
as it is the most often used operation.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References
 - https://logperiodic.com/rbsr.html
 - https://github.com/hoytech/negentropy