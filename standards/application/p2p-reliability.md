---
title: WAKU-P2P-RELIABILITY
name: Waku P2P Reliability
editor: Hanno Cornelius <hanno@status.im>
contributors: 
  - Danish Arora <danish@status.im>
  - Kaichao Sun <kaichao@status.im>
  - Oleksandr Kozlov <oleksandr@status.im>
  - Prem Chaitanya Prathi <prem@status.im>
---

## Abstract

This specification defines peer-to-peer (p2p) reliability within [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2) networks,
addressing dependability of message propagation between Waku node hops.
It situates the problem of p2p reliability between the lower layer _transport_ reliability
and the higher layer _end-to-end_ reliability.
We also propose strategies to enhance message propagation reliability
and mechanisms to detect and mitigate message loss
between Waku nodes,
taking into account the trade-offs between reliability, bandwidth usage, latency, and resource consumption.

## Reliability in a Waku context

Reliability can be considered on three layers within a [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2) context:

1. **Transport layer reliability:**
Reliability within the networking layer of an individual node.
This includes reliability of the underlying libp2p layer,
constituent transports,
peer management,
and peer discovery.
1. **Peer-to-peer (p2p) reliability:**
Reliability between nodes within a [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2) p2p network,
including message routing and discovery between peers.
Note that reliability at this layer is agnostic of the application.
At this layer, [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2) nodes have no knowledge of the origin or intended destination of messages being routed.
1. **End-to-end (e2e) reliability:**
Reliability within the application layer on top of the Waku p2p routing layer.
This layer is concerned with message delivery between intended participants from the application's perspective
regardless of the underlying Waku network.
As such, e2e reliability mechanisms are usually implemented within the encrypted application payload.

## Scope

This specification focuses on p2p reliability.
It does _not_ cover:
- Transport-level reliability mechanisms, such as fine-tuning libp2p parameters, improving peer discovery and management, etc.
- End-to-end message reliability protocols, covered by higher-layer application logic.

We focus on p2p reliability within the context of the following [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2) protocols:
- [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay): used by nodes to publish and receive messages on a pub/sub topic by participating in [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md) routing
- [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter): used by nodes to receive messages from a pub/sub topic without participating in gossipsub routing
- [WAKU2-LIGHTPUSH](../core/lightpush.md): used by nodes to publish messages to a pub/sub topic without participating in gossipsub routing
- [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store): used by any node to query and retrieve historical messages

## Problem statement

The primary challenge at the p2p layer is ensuring that messages sent from one Waku node propagate to peers
with a high probability of success
while minimising redundant transmissions and excessive bandwidth use.

Factors affecting reliability include:
- **Network instability**: detected and undetected network connectivity drops can lead to message loss or duplication.
- **Network churn**: frequent changes in the network topology as peers connect and disconnect can lead to potential message loss.
- **Node resource constraints**: nodes with limited bandwidth or processing power may struggle to handle high message volumes, leading to potential drops.
- **Adversarial behaviour**: adversarial nodes may deliberately drop, delay, or selectively forward messages.

## Reliability Strategies

### General

In this section, we first summarise strategies to improve p2p reliability
based on their functional effect before specifying the detailed strategies for each Waku protocol.
P2p reliability consists of either configuring individual Waku protocols
or composing several Waku protocols 
to achieve one or both of the following two objectives:

#### 1. Improve redundancy

Redundancy improves the probability that a message reaches its intended recipients.
Redundancy strategies can focus on:

- **Redundant publishing:**

Publishers MAY improve the probability of their messages propagating through the network
by publishing it simultaneously to several peers as first hop.
Several peers would then continue routing the message.
[17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) already implements redundant publishing by forwarding a message _at least_ to all mesh peers
or, as is RECOMMENDED for Waku, publishing the message to _all_ peers using [flood publishing](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#flood-publishing).
Configuring [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) is part of the transport layer and a detailed description falls outside the scope of this spec.
[WAKU2-LIGHTPUSH](../core/lightpush.md) MAY similarly be configured to publish to more than one lightpush service node at the same time.
See the [lightpush section](#lightpush) for a specification of the strategy.

- **Redundant receiving:**

Nodes MAY improve the probability for receiving all messages
by increasing the number of sources from which they receive messages.
[17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) already implements redundant receiving as messages are eagerly pushed from all mesh peers
with added gossip mechanisms to allow lazy pulling from all peers
as described in the [GossipSub v1.1 specification](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md).
Configuring [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) is part of the transport layer and a detailed description falls outside the scope of this spec.
[12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter) MAY similarly be configured to subscribe to receive messages from more than one filter service node at the same time.
See the [filter section](#filter) for a specification of the strategy.

#### 2. Detect and remedy losses

Nodes MAY combine different Waku protocols to detect and remedy possible losses.
Losses can occur either from the publisher's or the recipient's perspective.

- **Failure to publish:**

Nodes using either [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) or [WAKU2-LIGHTPUSH](../core/lightpush.md) to publish
MAY determine that a message has probably failed to publish
by combining these protocols with [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter) or [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store)
to attempt to retrieve the published message from other peers in the network.
Failure to detect the message could indicate a publishing failure.
The usual remedial action is to retransmit the message.
See [Store-based reliability](#store-based-reliability) for a specification of the strategy to combine [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) or [WAKU2-LIGHTPUSH](../core/lightpush.md) with [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store).
See [Lightpush](#lightpush) for a specification of the strategy to combine [WAKU2-LIGHTPUSH](../core/lightpush.md) with [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter).
Combining [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) with [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter) is conceptually possible, but underspecified.

- **Failure to receive:**

Nodes using either [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) or [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter) to receive messages
MAY determine message losses
by combining these protocols with [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store)
to compare their local message history with the historical messages cached in the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store).
The usual remedial action is to retrieve the missing messages from the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store).
See [Store-based reliability](#store-based-reliability) for a specification of this strategy.

### Store-based reliability

[13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) provides a way for nodes to query the existence of or fetch specific historical messages.
Nodes using any combination of [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay), [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter) and [WAKU2-LIGHTPUSH](../core/lightpush.md) to publish/receive messages
MAY combine these protocols with [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) to improve p2p reliability.
Depending on the use case, such a node MAY use any of the strategies below:

#### 1. Store-based reliability for publishing

- A publishing node using this strategy MUST consider a published message as "unacknowledged" at first.
- The publisher MUST keep a copy of this message against its [deterministic message hash](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing) in a local outgoing message buffer.
It MAY keep the last several published messages in this buffer.
- The publisher MUST periodically perform a [presence query](https://rfc.vac.dev/waku/standards/core/13/store#presence-queries) to the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) service
against the message hashes of all published messages in the outgoing buffer
to verify their existence in the store.
    - If a message hash exists in the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) service,
the publisher SHOULD consider the corresponding message as "acknowledged" and remove it from the outgoing buffer.
    - If a message hash does not exist in the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) service,
the publisher MUST consider the corresponding message as still "unacknowledged".
- The publisher SHOULD retransmit all unacknowledged messages either periodically or upon reception of the presence query response
until a positive presence query response is received for the corresponding message hash.
A publisher MAY consider a message publication as having failed irremediably after a set number of failed presence query attempts.

#### 2. Store-based reliability for receiving

- A receiving node using this strategy MUST have a local cache of the [deterministic message hashes](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing) of all received messages
spanning _at least_ the time period for which reliability is required.
- The node MUST periodically perform a [content filtered query](https://rfc.vac.dev/waku/standards/core/13/store#content-filtered-queries) to the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) service,
spanning the time period for which reliability is required ("reliability window")
and including all content topics over which it is interested to receive messages.
The `include_data` field SHOULD be set to `false` to retrieve only the matching message hashes from the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) service.
If a connection loss is detected (e.g. if a query fails due to disconnection),
the next query MAY span at least the time period since the last successful query
if this is longer than the reliability window.
- The node MUST compare the received message hashes to those in the local cache.
Any message hashes in the response that are not in the local cache MUST be considered "missing".
- The node SHOULD perform a [message hash lookup query](https://rfc.vac.dev/waku/standards/core/13/store#message-hash-lookup-queries) for all missing message hashes
to retrieve the full contents of the corresponding messages.
It MAY do so either periodically (in batches) or upon reception of the content filtered query response.
- The node MUST add to the local cache all message hashes corresponding to the retrieved messages,
in order to prevent them from being considered missing in future.

#### 3. Combined store-based reliability for publishing and receiving messages

- A node using this strategy MUST have a local cache of the [deterministic message hashes](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing) of all received messages
spanning _at least_ the time period for which reliability is required.
- In addition, the node MUST consider a published message as "unacknowledged" at first. The node MUST keep a copy of this message against its [deterministic message hash](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing) in a local outgoing message buffer.
It MAY keep the last several published messages in this buffer.
- The node MUST periodically perform a [content filtered query](https://rfc.vac.dev/waku/standards/core/13/store#content-filtered-queries) to the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) service,
spanning the time period for which reliability is required
and including all content topics over which it is interested to both publish and receive messages.
The `include_data` field SHOULD be set to `false` to retrieve only the matching message hashes from the [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) service.
- The node MUST compare the message hashes in the content filtered query response to the message hashes of all published messages in the outgoing buffer
to verify their existence in the store.
    - If a message hash is included in the query response,
the publisher SHOULD consider the corresponding message as "acknowledged" and remove it from the outgoing buffer.
    - If a message hash is not included in the query response,
the publisher MUST consider the corresponding messages as still "unacknowledged".
- In addition, the node MUST compare the message hashes in the content filtered query response to the message hashes in the local cache.
Any message hashes in the response that are not in the local cache MUST be considered "missing".
- The node SHOULD retransmit all unacknowledged messages in the outgoing buffer,
either periodically or upon reception of the query response,
until a positive inclusion in a follow-up query response for the corresponding message hashes.
The node MAY consider a message publication as having failed irremediably after a set number of query attempts without inclusion.
- In addition, the node SHOULD perform a [message hash lookup query](https://rfc.vac.dev/waku/standards/core/13/store#message-hash-lookup-queries) for all missing message hashes
to retrieve the full contents of the corresponding messages.
It MAY do so either periodically (in batches) or upon reception of the content filtered query response.
- The node MUST add to the local cache all message hashes corresponding to the retrieved messages,
in order to prevent them from being considered missing in future.

### Lightpush

[WAKU2-LIGHTPUSH](../core/lightpush.md) provides a way for client nodes to publish messages to a pub/sub topic via a lightpush service node without participating in gossipsub routing.
Lightpush clients MAY use any of the following strategies to improve reliability of the service:

#### 1. Maintain a pool of reliable lightpush service nodes

A lightpush client using this strategy MUST maintain a pool of reliable lightpush service nodes.
[Discovery](https://rfc.vac.dev/waku/standards/core/10/waku2#discovery-domain) of these service nodes falls outside the scope of this specification.
As a simple heuristic, a lightpush client MAY consider all discovered lightpush service nodes as reliable until it detects a service failure.
In case the lightpush service fails due to service node behaviour,
the client MAY disconnect from the service node and replace it with another service node from the pool.
Such a client SHOULD also remove the failing service node from the pool of reliable service nodes.

We RECOMMEND replacing a lightpush service node after a single failure in the following categories:
- the connection to the service node is lost or a lightpush request times out
- the lightpush request fails due to a service-side error, for example if the response contains one of the following [error codes](../core/lightpush.md#examples-of-possible-error-codes):
    - `UNSUPPORTED_PUBSUB_TOPIC`
    - `INTERNAL_SERVER_ERROR`
    - `NO_PEERS_TO_RELAY`
- the request failed without an error response

#### 2. Redundant lightpush publishing

A lightpush client using this strategy MUST publish each message simultaneously to two or more lightpush service nodes.
Note that bandwidth usage increases proportionally to the amount of service nodes used.
For this reason, we RECOMMEND using only two lightpush service nodes at a time.

#### 3. Retransmit on failure

- A lightpush client using this strategy MUST wait for a [WAKU2-LIGHTPUSH response](../core/lightpush.md) after publishing a message.
- If the response times out or contains [a recoverable error code](../core/lightpush.md#examples-of-possible-error-codes) (e.g. `TOO_MANY_REQUESTS`)
the lightpush client SHOULD attempt to retransmit the message after some interval.
- The client MAY choose to continue retransmitting the message until an `OK` response is received from the service node.
The interval between each retransmission attempt is up to the implementation,
but we RECOMMEND starting with `1 second` and increasing it after each failure.
- The client MAY consider a message publication as having failed irremediably after a set number of failed lightpush requests.

#### 4. Retransmit on loss detection

> *_Note:_* Lightpush clients participating in [Store-based reliability](#store-based-reliability) already performs this strategy and can ignore this section.

- A lightpush client using this strategy MUST use either [Store-based reliability](#store-based-reliability)
or install one or more [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter) subscriptions matching the content topic(s) used for publishing.
In this way, the client can confirm that a published message did indeed reach the targeted store or filter service node(s).
- The store queries or filter subscription SHOULD be requested at service nodes different from those used for the lightpush service.
- If the client determines that a published message has not been received by the filter or store service node,
it SHOULD retransmit the message after some interval.
- The client MAY choose to continue retransmitting the message until it is confirmed by one more service nodes.
The interval between each retransmission attempt is up to the implementation,
but we RECOMMEND starting with `1 second` and increasing it after each attempt.
- The client MAY consider a message publication as having failed irremediably after a set number of failed lightpush requests.

### Filter

[12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter) provides a way for client nodes to receive messages from a pub/sub topic via a filter service node without participating in gossipsub routing.
Filter clients MAY use any of the following strategies to improve reliability of the service:

#### 1. Maintain a pool of reliable filter service nodes

A filter client using this strategy MUST maintain a pool of reliable filter service nodes.
[Discovery](https://rfc.vac.dev/waku/standards/core/10/waku2#discovery-domain) of these service nodes falls outside the scope of this specification.
As a simple heuristic, a filter client MAY consider all discovered filter service nodes as reliable until it detects a service failure.
In case the filter service fails due to service node behaviour,
the client MAY disconnect from the service node and replace it with another service node from the pool.
Such a client SHOULD also remove the failing service node from the pool of reliable service nodes.

We RECOMMEND replacing a filter service node under the following conditions:
- a [`SUBSCRIBE`](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscribe) request fails due to a service-side error
- a [`SUBSCRIBER_PING`](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscriber_ping) request fails twice in a row

#### 2. Redundant filter subscriptions

A filter client using this strategy MUST subscribe to two or more filter service nodes.
Such clients SHOULD filter out duplicate messages by comparing [deterministic message hashes](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing).
Note that both bandwidth usage and computational complexity increases proportionally to the amount of service nodes used.
For this reason, we RECOMMEND using only two filter service nodes at a time.

#### 3. Maintaining healthy subscriptions

- A filter client using this strategy MUST regularly send a [`SUBSCRIBER_PING`](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscriber_ping) to each of its filter service nodes,
to ensure that the service node is online and maintaining an active subscription for the client.
The interval between each `SUBSCRIBER_PING` is up to the implementation,
but we RECOMMEND `1 minute`.
- If the `SUBSCRIBER_PING` request times out or returns an error code,
the client SHOULD attempt to reinstall its filter subscription with a [`SUBSCRIBE`](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscribe) request.
- If a [`SUBSCRIBER_PING`] or [`SUBSCRIBE`] request fails more than once to the same filter service node,
the client MAY choose to replace this service node with another from the service node pool.
We RECOMMEND a strict policy of replacing filter service nodes after only two `SUBSCRIBER_PING` failures
or after a single `SUBSCRIBE` failure.
- A filter client MAY also choose to refresh its existing subscriptions periodically,
by submitting the same filter criteria as before in a new [`SUBSCRIBE`](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscribe) request to the same service node.
This helps ensure that local and remote views of filter criteria remains synchronised.

#### 4. Store query on loss detection

> *_Note:_* Filter clients participating in [Store-based reliability](#store-based-reliability) already performs this strategy and can ignore this section.

- A filter client using this strategy MUST use [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store) queries to retrieve lost messages.
- Such clients MAY use [Store-based reliability](#store-based-reliability) to periodically detect and remedy message losses.
- If [Store-based reliability](#store-based-reliability) is unsuitable (e.g. due to the high resource usage of repeated store queries),
the client MAY perform opportunistic store queries covering periods over which it detected a disconnection.
For example, a client MAY consider itself offline over a period of repeated failed regular pings and perform a store query once the connection has been restored.
The store query SHOULD cover the period from the last successfully received message to the reestablishment of connectivity.

## Tradeoffs

All p2p reliability strategies set out in this document increases resource usage,
most prominently bandwidth usage but also processing power and storage in some circumstances.
As such, each strategy SHOULD be carefully considered and configured based on the intended use case.
Increasing redundancy 20-fold may significantly improve reliability,
but the accompanying bump in resource usage will be unacceptable for the average Waku user.
At the same time,
none of the reliability strategies described here can _guarantee_ end-to-end reliability from an application's perspective.
This is due to the inherent probabilistic nature of p2p message propagation on the routing layer.
For example, certain sections of the network may become temporarily unreachable due to a network split,
without this being visible on a hop-to-hop basis.
Only the application has the end-to-end view to ensure reliability spanning all routing layer hops between that application's publishers and intended recipients.
For applications with an integrated end-to-end reliability protocol,
most p2p reliability strategies can be minimally configured (or even disabled) to save resources.

## References

1. [10/WAKU2](https://rfc.vac.dev/waku/standards/core/10/waku2)
1. [12/WAKU2-FILTER](https://rfc.vac.dev/waku/standards/core/12/filter)
1. [13/WAKU2-STORE](https://rfc.vac.dev/waku/standards/core/13/store)
1. [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay)
1. [14/WAKU2-MESSAGE](https://rfc.vac.dev/waku/standards/core/14/message#deterministic-message-hashing)
1. [GossipSub v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#flood-publishing)
1. [WAKU2-LIGHTPUSH](../core/lightpush.md)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).