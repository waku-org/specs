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
in the context of keeping syncronized two Waku Store nodes.
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
Nodes enabling Waku Sync SHOULD have the relay and store protocols enabled and
keep messages for at least the last hour. TODO do we need to say this, sounds like an impl. detail

After each sync session, nodes SHOULD use Store queries to acquire missing messages.

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
by sending a `SyncPayload` with only the `negentropy` field set.
This field MUST contain the first negentropy payload created by the client for this session.

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
an empty `engentropy` field.

### Storage Pruning
TODO? do we need to talk about that, seams like an implementation detail

### Message transfer
TODO? do we need to talk about that, seams like an implementation detail

# Attack Vectors
Nodes using `WAKU-SYNC` are fully trusted.
Message hashes are assumed to be of valid messages received via Waku Relay.

Further refinements to the protocol are planned
to reduce the trust level required to operate.
Notably by verifing message RLN proofs at reception.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References
 - https://logperiodic.com/rbsr.html
 - https://github.com/hoytech/negentropy