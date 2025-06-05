---
title: WAKU-STORE-SYNC
name: Waku Store Synchronization
editor: Simon-Pierre Vivier <simvivier@status.im>
contributors:
---

## Abstract

This document describes a strategy to keep [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) nodes synchronised,
using a combination of [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) queries 
and the [WAKU-SYNC](../core/sync.md) protocol.

## Background / Rationale / Motivation

Message propagation in [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2) networks is not perfect.
Even with [peer-to-peer reliability](../application/p2p-reliability.md) mechanisms,
a certain amount of routing losses are always expected between Waku nodes.
For example, nodes could experience brief, undetected disconnections,
undergo restarts in order to update software,
or suffer losses due to resource constraints.

Whatever the source of the losses,
this affects applications and services relying on the message routing layer.
One such service is the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) protocol
that allows nodes to cache historical [14/WAKU2-MESSAGE](https://rfc.vac.dev/waku/standards/core/14/message)s from the routing layer,
and provision these to clients.
Using Waku Store Sync,
[13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) can remain synchronised
and reach eventual consistency despite occasional losses on the routing layer.

## Scope:

Waku Store Sync aims to provide a way for [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) nodes
to compare and retrieve differences with other [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) nodes,
in order to remedy messages that might have been missed or lost on the routing layer.

It seeks to cover the following loss scenarios:
1. Short-term offline periods, for example due to a restart or short-term node maintenance
2. Occasional message losses that occur during normal operation, due to short-term instability, churn, etc.

For the purposes of this document,
we define short-term offline periods as no more than `1` hour
and occasional message losses as no more than `20%` of total routed messages.

It does not aim to address recovery after long-term offline periods,
or to address massive message losses due to extraordinary circumstances,
such as adversarial behaviour.
Although Store Sync could perhaps work in such cases,
it's not optimised or designed for catastrophic loss recovery.
Large scale recovery falls beyond the scope of this document.
We provide further recommendations for reasonable parameter defaults below.

## Theory / Semantics

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

A [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) node with Store Sync enabled:
1. MAY use [Store Resume](#store-resume) to recover messages after detectable short-term offline periods
2. MUST use [Waku Sync](#waku-sync) to maintain consistency with other nodes and recover occasional message losses

### Store Resume

Store Sync nodes MAY use Store Resume to fill the gap in messages for any short-term offline period.
Such a node SHOULD keep track of its last online timestamp.
It MAY do so by periodically storing the current timestamp on disk while online.
After a detected offline period has been resolved,
or at startup,
a Store Sync node using Store Resume SHOULD select another [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) node using any available discovery mechanism.
We RECOMMEND that this to be a random node.
Next, the Store Sync node SHOULD perform a [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store#content-filtered-queries) query
to the selected node for the time interval since it was last online.
Messages returned by the query are then added to the local node storage.
It is RECOMMENDED to limit the time interval to a maximum of `6` hours.

### Waku Sync

Even while online, Store Sync nodes may occasionally miss messages.
To remedy any such losses and to achieve eventual consistency,
Store Sync nodes MUST mount [WAKU2-SYNC](./sync.md) protocol
to detect and exchange differences with other Store Sync nodes.
As described in that specification,
[WAKU2-SYNC](./sync.md) consists of two sub-protocols.
Both sub-protocols MUST be used by Store Sync nodes in the following way:
1. `reconciliation` MUST be used to detect and exchange differences between [14/WAKU2-MESSAGE](https://rfc.vac.dev/waku/standards/core/14/message)s cached by the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) node
2. `transfer` MUST be used to transfer the actual content of such differences.
Messages received via `transfer` MUST be cached in the same archive backend
where the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) node caches messages received via normal routing.

#### Periodic syncing

Store Sync nodes SHOULD periodically trigger [WAKU2-SYNC](./sync.md).
We RECOMMEND syncing at least once every `5` minutes with `1` other Store Sync peer.
The node MAY choose to sync more often with more peers
to achieve faster consistency.
Any peer selected for Store Sync SHOULD be chosen at random.

Discovery of other Store Sync peers falls outside the scope of this document.
For simplicity, a Store Sync node MAY assume that any other [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) peer
supports Store Sync and attempt to trigger a sync operation with that node.
If the sync operation then fails (due to unsupported protocol),
it could continue attempting to sync with other [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) peers on a trial-and-error basis
until it finds a suitable Store Sync peer.

#### Sync window

For every [WAKU2-SYNC](./sync.md) operation,
the Store Sync node SHOULD choose a reasonable window of time into the past
over which to sync cached messages.
We RECOMMEND a sync window of `1` hour into the past.
This means that the syncing peers will compare
and exchange differences in cached messages up to 1 hour into the past.
A Store Sync node MAY choose to sync over a shorter time window to save resources and sync faster.
A Store Sync node MAY choose to sync over a longer time window to remedy losses over a longer period.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


## References

- [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2)
- [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store)
- [14/WAKU2-MESSAGE](https://rfc.vac.dev/waku/standards/core/14/message)
- [WAKU-P2P-RELIABILITY](../application/p2p-reliability.md)
- [WAKU2-SYNC](./sync.md)