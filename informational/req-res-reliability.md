---
title: REQ-RES-RELIABILITY
name: Request-response protocols reliability
category: Best Current Practice
tags: [informational]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
---

## Abstract
This RFC describes set of instructions used across different [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md?plain=1#L3) implementations for improved reliability in request-response protocols such as [WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/19/lightpush.md?plain=1#L3C11-L3C26) and [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md?plain=1#L3).

## Motivation

Descriptions of mentioned protocols do not define some of the real world use cases that are oftenly observed in unreliable network environment. Such use cases can be: recovery from offline state, decrease rate of missed messages, increase probability of messages being broadcasted within the network.

## Suggestions

### Node health

As a useful metric to define and implement for determining quality of provided service by a node:
- unhealthy - no peer connections are available regardless of protocol;
- minimally healthy:
  - Relay has less than 4 peers connected;
  - Filter and LightPush has one per each peer connection available;
- sufficiently healthy:
  - Relay has minimum 4 peers connected;
  - more than 1 connection in Filter and at least 2 connections available in LightPush;

### Peers and connection management

- Each protocols should retain a pool of reliable peers. In case a protocol failed to use any peer more than once - connection to it should be dropped and new peer should be added to the pool instead. 

- During discovery of new peers it is better to filter unwonted out based on ENR / multiaddress. For example in some cases `circuit-relay` addresses are not needed when we try to find and connect to peers directly.

- When peer is discovered second time, we need to be sure to keep connection information up to date in Peer Store.

### Light Push 

In case sending message failed - node should try to re-send message after some interval. 

### Filter

- To decrease chances of missing messages a node can initiate more than one subscription through Filter protocol to the same content topic and filter out duplicates. This will increase bandwidth consumption and would depend on the information exchanged under content topic in use.

- In case a node goes offline while having an active subscription - it is important to do ping again right after node appears online. In case ping fails - re-subscribe request should be fired to a new peer.

- While registering Filter subscriptions - it is advised to batch requests for multiple content topics into one in order to reduce amount of queries sent to a node. 

- During creation of a new subscription it can be benefitial to use only new peers to which no subscriptions yet present and not use peers with which Filter already failed.

## Security/Privacy Considerations

If there are none, this section MAY state that fact.
This section MAY contain additional relevant information, e.g. an explanation as to why there are no security consideration for the respective document.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

A list of references.
