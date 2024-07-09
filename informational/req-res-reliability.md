---
title: REQ-RES-RELIABILITY
name: Request-response protocols reliability
category: Best Current Practice
tags: [informational]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
- Prem Chaitanya Prathi <prem@status.im>
- Danish Arora <danish@status.im>
---

## Abstract
This RFC describes set of instructions used across different [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md) implementations for improved reliability during usage of request-response protocols by a light node:
- [WAKU2-LIGHTPUSH](../standards/core/lightpush.md) - is used for sending messages;
- [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) - is used for receiving messages;

### Definitions
- Service node - provides services to other nodes such as relaying messages send by LightPush to the network or service messages from the network through Filter, usually serves responses;
- Light node - connects to and uses one or more service nodes via LightPush and/or Filter protocols, usually sends requests;
- Service node failure - can mean various things depending on the protocol in use:
  - generic protocol failure - request is timed out or failed without error codes;
  - LightPush specific failure - refer to [error codes](../standards/core/lightpush.md#examples-of-possible-error-codes) and consider request a failure when it is clear that service node cannot serve any future request, for example when service node does not have any peers to relay and returns `NO_PEERS_TO_RELAY`;
  - Filter specific failure - we consider service node failing when it cannot serve [subscribe](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscribe) or [ping](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscriber_ping) request with OK status;

## Motivation

Specifications of the mentioned protocols do not define some of the real world use cases that are often observed in unreliable network environment from the perspective of light nodes that are consumers of LightPush and/or Filter protocols.
Such use cases can be: recovery from offline state, decrease rate of missed messages, increase probability of messages being broadcasted within the network, unreliability of the service node in use.

## Suggestions

### Node health

Node health is a metric meant to determine the connectivity state of a light node and its present ability to reliably send and receive messages from the network.
We consider this reliability to be dependant on amount of simultaneous connections to responsive service nodes.
Unfortunately the more connections light node establishes - the more bandwidth is consumed.
To address this we suggest following states:
- unhealthy - no connections to service nodes are available regardless of protocol;
- minimally healthy:
  - Filter has one service node connection;
  - LightPush protocol has one service node connection;
- sufficiently healthy:
  - Filter has at least 2 connections available to service nodes;
  - LightPush has at least 2 connections available to service nodes;

### Peers and connection management

#### Pool of reliable service nodes
Light nodes should maintain a pool of reliable service nodes for each protocol.
In case service node [fails](./req-res-reliability.md#definitions) to serve protocol request - 
light node should drop connection to it and a new service node should be connected and added to the pool instead.

We advice to replace service node for LightPush right after first failure in case:
- connection to it is lost or request timed out;
- it's response contains [error codes](../standards/core/lightpush.md#examples-of-possible-error-codes): `UNSUPPORTED_PUBSUB_TOPIC`, `INTERNAL_SERVER_ERROR` or `NO_PEERS_TO_RELAY`;
- request failed but without error message returned;

For Filter we'd recommend replacing service node:
- [request for subscription](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscribe) so it cannot be initiated;
- [ping](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscriber_ping) failed 2 times in a row;

#### Selection of discovered service nodes
During discovery light node should filter out service nodes based on preferences before establishing connection.
These preferences might include:
- [Libp2p multiadresses](https://github.com/libp2p/specs/blob/master/addressing/README.md) of a service node;
- Waku or libp2p protocols that a service node implements;
- Wakus shards that a service node is part of;

More details about discovery can be found at [WAKU2 Discovery domain](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md#discovery-domain) or [RELAY-SHARDING Discovery](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md#discovery).

Examples of filtering:
- When light node discovers service nodes that implement needed Waku protocols - it should prioritize those that implement most recent version of protocol;
- Light node must connect only to those service nodes that participate in needed shard and cluster;
- Light node must use only those service nodes that implement needed transport protocols;
- When [Circuit V2](https://github.com/libp2p/specs/blob/master/relay/circuit-v2.md) multi-addresses discovered by a light node - it should prefer other service nodes that can be connected directly if possible;

#### Continuous discovery
Light nodes must keep information about service nodes up to date.
For example when a service node is discovered second time,
we need to be sure to keep connection information up to date in Peer Store.

Information that is important to be up to date:
- [ENR](../standards/core/enr.md) information;
- [Libp2p multiaddresses](https://github.com/libp2p/specs/blob/master/addressing/README.md);

### LightPush

#### Sending with redundancy
To improve chances of delivery of messages light node can attempt sending same message via LightPush to 2 or more service nodes at the same time.
While doing so it is important to note that bandwidth consumption increases proportionally to amount of additional service nodes used.
Our advice to use 2 service nodes at a time.

#### Retry on failure
When light node sends a message it must await for LightPush response from service node and check it for [possible error codes](../standards/core/lightpush.md#examples-of-possible-error-codes).
In case request failed without error code or response contains errors that can be temporary for service node (e.g `TOO_MANY_REQUESTS`) - 
light node should try to re-send message after some interval and continue doing so until OK response is received or canceled.
Interval time can be arbitrary but we recommend starting with 1 second and increasing it on each failure during LightPush send.
Important to note that [per another recommendation](./req-res-reliability.md#pool-of-reliable-service-nodes) - light node should replace failing service node with another within pool of service nodes used by LightPush.

#### Retry missing messages
Light node can verity that network that is used at the moment has seen messages that were sent via LightPush earlier.
In order to do that light node should use [Store protocol](../standards/core/store.md) or [Filter protocol](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) to a different than the one used for LightPush service node.

By using Store protocol light node can query any service node that implements Store protocol and see if the messages that were sent in the past period were seen.
Due to [Store message eligibility](https://github.com/waku-org/specs/blob/master/standards/core/store.md#waku-message-store-eligibility) only some of the messages will be stored so there is a limit as to which messages can be verified by Store queries.
Our advice to do periodic Store queries once per 30 seconds.

By using Filter protocol's active [subscription](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#filter-push) light node can verify that message that was sent through LightPush was seen by another service node in the network.
Filter protocol does not have such limitation as to type of messages received with subscription 
but active subscription does not allow to see messages exchanged in the network while light node was offline.

In case some of the messages were not verified by any of the previous methods  - they should be re-sent by LightPush using different service node.

### Filter

#### Regular pings
To ensure that subscription is maintained by a service node and not closed - light node should do recurring [pings](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscriber_ping).
Our advice for light node to send ping requests once per minute.
In case light node does not receive OK response or it times out 2 times - such service node should be replaced as part of maintenance of [pool of reliable service nodes](./req-res-reliability.md#pool-of-reliable-service-nodes).
Right after such replace light node must create new subscription to newly connected service node as described in [Filter specification](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md).

#### Redundant subscriptions for message loss mitigation
To mitigate possibility of messages not being delivered by a service node - we advice to consider using multiple Filter subscriptions. 
Light node can initiate two subscriptions to the same content topic but to different service nodes. 
While receiving messages through two subscriptions - duplicates must be dropped by using [deterministic message hashing](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md#deterministic-message-hashing).
Note that such approach increase bandwidth consumption proportionally to amount of extra subscriptions established and should be used with caution. 

#### Offline recoverability
Network state should be monitored by light node and in case it goes offline - [regular pings](./req-res-reliability.md#regular-pings) must be stopped. 
When network connection returns light node should initiate [Filter ping](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md#subscriber_ping) to service nodes in use.
In case those pings fail light node must replace service nodes following advice of [pool of reliable service nodes](./req-res-reliability.md#pool-of-reliable-service-nodes) without waiting for multiple failures.

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
