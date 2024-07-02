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
This RFC describes set of instructions used across different [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md) implementations for improved reliability during usage of request-response protocols by a light node.
Such protocols are:
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

As a useful metric to define and implement for determining quality of provided service by a node:
- unhealthy - no peer connections are available regardless of protocol;
- minimally healthy:
  - Relay has less than 4 peers connected;
  - Filter and LightPush clients has one per each peer connection available;
- sufficiently healthy:
  - Relay has minimum 4 peers connected;
  - more than 1 connection in Filter and at least 2 connections available in LightPush;

### Peers and connection management

- Each protocols should retain a pool of reliable peers. In case a protocol failed to use any peer more than once - connection to it should be dropped and new peer should be added to the pool instead. 

- During discovery of new peers it is better to filter out based on ENR / multiaddress. For example in some cases `circuit-relay` addresses are not needed when we try to find and connect to peers directly.

- When peer is discovered second time, we need to be sure to keep connection information up to date in Peer Store.

### Light Push 

- While sending message with Light Push - it is advised to use more than 1 peer in order to increase chances of delivering message.

- If sending message is failed to all of the peers - node should try to re-send message after some interval and continue doing so until OK response is received. 

### Filter

- To decrease chances of missing messages a node can initiate more than one subscription through Filter protocol to the same content topic and filter out duplicates. This will increase bandwidth consumption and would depend on the information exchanged under content topic in use.

- In case a node goes offline while having an active subscription - it is important to do ping again right after node appears online. In case ping fails - re-subscribe request should be fired to a new peer.

- While registering Filter subscriptions - it is advised to batch requests for multiple content topics into one in order to reduce amount of queries sent to a node. 

- During creation of a new subscription it can be beneficial to use only new peers to which no subscriptions yet present and not use peers with which Filter already failed.

## Security/Privacy Considerations

None of the mentioned recommendations incur privacy or security tradeoffs and in some cases increase k-anonymity (e.g having unique peers for Filter subscriptions).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
