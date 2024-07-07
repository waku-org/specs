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
- [WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/19/lightpush.md) - is used for sending messages;
- [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) - is used for receiving messages;

### Definitions
- Service node - provides services to other nodes such as relaying messages send by LightPush to the network or broadcasts messages from the network through Filter, usually serves responses;
- Light node - connects to and uses one or more service nodes via LightPush and/or Filter protocols, usually sends requests;

## Motivation

Specifications of the mentioned protocols do not define some of the real world use cases that are often observed in unreliable network environment from the perspective of light nodes that are consumers of LightPush and/or Filter protocols.
Such use cases can be: recovery from offline state, decrease rate of missed messages, increase probability of messages being broadcasted within the network.

## Suggestions

### Node health

Node health is a metric meant to determine how reliable a light node is.
We consider this reliability to be dependant on amount of simultaneous connections to responsive service nodes.
Unfortunately the more connections light node establishes - the more bandwidth is consumed.
To address this we suggest following metrics:
- unhealthy - no connections to service nodes are available regardless of protocol;
- minimally healthy:
  - Filter has one service node connection;
  - LightPush protocol has one service node connection;
- sufficiently healthy:
  - Filter has at least 2 connections available to service nodes;
  - LightPush has at least 2 connections available to service nodes;

### Peers and connection management

#### Pool of reliable service nodes
Light node should maintain a pool of reliable service nodes for each protocol.
In case service node fails to serve protocol request from a light node 3 times - light node should drop connection to it and a new service node should be connected and added to the pool instead.

#### Seection of discovered service nodes
During discovery light node should filter out service nodes based on preferences before establishing connection.
These preferences might include:
- [Libp2p multiadresses](https://github.com/libp2p/specs/blob/master/addressing/README.md) of a service node;
- Waku or libp2p protocols that a service node implements;
- Wakus shards that a service node is part of;

More details about discovery can be found at [WAKU2 Discovery domain](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md#discovery-domain) or [RELAY-SHARDING Discovery](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md#discovery).

Examples of filtering:
- When [Circuit V2](https://github.com/libp2p/specs/blob/master/relay/circuit-v2.md) multiaddresses discovered by a light node - it might avoid connecting such service nodes and wait for service nodes that can be connected directly;
- When light node discovers service nodes that implement needed Waku protocols - it should prioritize those that implement most recent version of protocol;
- Light node must connect only to those service nodes that participate in needed shard;

#### Continuous discovery
Light node must keep information about service nodes up to date.
For example when a service node is discovered second time,
we need to be sure to keep connection information up to date in Peer Store.

### LightPush

#### Sending with redundancy
To improve chances of delivery of messages light node can attempt sending same message via LightPush to 2 or more service nodes at the same time.
While doing so it is important to note that bandwidth consumption increases.

#### Retry on failure
When light node sends a message it must await for LightPush response from service node and check it for [possible error codes](../standards/core/lightpush.md#examples-of-possible-error-codes).
In case request failed without error code or response contains errors that can be temporary for service node (e.g `TOO_MANY_REQUESTS` or `NO_PEERS_TO_RELAY`) - 
light node should try to re-send message after some interval and continue doing so until OK response is received or canceled.
Interval time can be arbitrary but we recommend starting with 1 second and increasing it on each failure during LightPush send.
Important to note that [per another recommendation](./req-res-reliability.md#pool-of-reliable-service-nodes) - light node should replace failing service node with another within pool of service nodes used by LightPush.

#### Retry missing messages
Light node can verity that network that is used at the moment has seen messages that were sent via LightPush earlier.
In order to do that light node should use [Store protocol](../standards/core/store.md).
By using Store light node can query service node that implements Store and see if the messages that were sent in the past period were seen.

### Filter

- To decrease chances of missing messages a node can initiate more than one subscription through Filter protocol to the same content topic and filter out duplicates. This will increase bandwidth consumption and would depend on the information exchanged under content topic in use.

- In case a node goes offline while having an active subscription - it is important to do ping again right after node appears online. In case ping fails - re-subscribe request should be fired to a new peer.

- While registering Filter subscriptions - it is advised to batch requests for multiple content topics into one in order to reduce amount of queries sent to a node. 

- During creation of a new subscription it can be beneficial to use only new peers to which no subscriptions yet present and not use peers with which Filter already failed.

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).