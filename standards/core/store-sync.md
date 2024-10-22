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
By having store nodes syncronize with each other through various protocols,
the set of archived messages network wide will be eventually consistent.

## Theory / Semantics

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Various protocols and features that help with message consistency are described below.

### Store Resume

<!-- restating this as a heuristic rather than a high-level description of functionality allows the spec to be useful for new implementers -->

This feature allow a node to fill the gap in messages for the period it was last offline.
At startup, a node use the Store protocol to query a random node for
the time interval since it was last online.
Messages returned by the query are then added to the node local archive storage.
It is RECOMMENDED to limit the time interval to a maximum of 6 hours.

### Waku Sync

<!--
For the new version. "think of a specification as a heuristic/set of directives that almost form a kind of pseudocode. This becomes easier if you use the RFC keywords for each directive"
-->

Nodes that stay online can still miss messages.
[Waku Sync](https://github.com/waku-org/specs/blob/master/standards/core/sync.md) is the protocol used to find those message hashes by periodically syncing with random nodes.
It is RECOMMENDED to trigger a sync with a random peer every 5 minutes for a time range of the last hour.
After both message hash sets are reconciled,
the Store protocol SHOULD be used to reciprocally query the other node with the message hash differences found.
Messages returned by the query MUST be added to the node local Waku sync and archive storage.

<!-- TODO add the number of node to sync with -->

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


## References

- [Waku Sync](https://github.com/waku-org/specs/blob/master/standards/core/sync.md)