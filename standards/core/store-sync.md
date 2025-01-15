---
title: WAKU-STORE-SYNC
name: Waku Store Synchronization
editor: Simon-Pierre Vivier <simvivier@status.im>
contributors:
---

## Abstract

This document describe the strategy Waku Store node will employ to stay synchronized.
The goal being that all store nodes eventually archive the same set of messages.

## Background / Rationale / Motivation

Message propagation in the network is not perfect,
even with GossipSub mechanisms to detect missed messages.
Nodes can also go offline for various reason outside our control.
Store nodes that want to provide a good service must be able to remedy situations like these.
By having store nodes synchronize with each other through various protocols,
the set of archived messages network wide will be eventually consistent.

## Theory / Semantics

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Various protocols and features that help with message consistency are described below.

### Store Resume

This feature allow a node to fill the gap in messages for the period it was last offline.
At startup, a node SHOULD use the Store protocol to query a random node for
the time interval since it was last online.
Messages returned by the query are then added to the local node storage.
It is RECOMMENDED to limit the time interval to a maximum of 6 hours.

### Waku Sync

Nodes that stay online can still miss messages.
[Waku Sync](https://github.com/waku-org/specs/blob/master/standards/core/sync.md) is 2 libp2p protocols used to find those messages and mend the differences by periodically syncing with random nodes.
It is RECOMMENDED to trigger a sync with a random peer that supports the protocols every 5 minutes for a time range of the last hour.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


## References

- [Waku Sync](https://github.com/waku-org/specs/blob/master/standards/core/sync.md)