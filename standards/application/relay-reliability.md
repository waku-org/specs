---
title: RELIABILITY-RELAY
name: Reliability for Relay Protocol
category: (Standards Track|Informational|Best Current Practice)
tags: [reliability, application]
editor: Kaichao Sun <kaichao@status.im>
contributors:
---

## Abstract

[Relay Protocol](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) in Waku is efficient for routing messages, but there's no guarantee that a message will reach to its destination, for example, the receiver in a chat application. In general, a message in Waku network includes 3 status:

- outgoing, the message is posted by its creator via the relay protocol
- sent, the message is received by any other node in the network
- delivered, the message is acknowledged by the receiver

Application like Status already uses [MVDS](https://github.com/vacp2p/rfc-index/blob/main/vac/2/mvds.md) for e2e acknowledgement in direct messages and group chat. There is an ongoing [discussion](https://forum.vac.dev/t/end-to-end-reliability-for-scalable-distributed-logs/293/13) about a more general and bandwidth efficient solution for e2e reliablity.

Before we have a complete design for e2e reliability, we need to compose existing protocols to increase the reliability of the relay protocol. This document proposes a few options for such composition.

## Motivation

The [store protocol](https://github.com/waku-org/specs/blob/master/standards/core/store.md) provides a way for nodes in the network to query the existence of messages or fetch specific messages based on the search criteria.

**Search criteria with message hash**

The node may have connection issues to *publish* messages via relay network. This search criteria can be used to check whether a message is populated in the network or not. The message exists in store node can be marked from `outgoing` to `sent` by application. If the message is not found in the store node, the application can resend the message.

**Search criteria with topics and time range**

The node may have connection issues to *receive* messages via relay network. This search criteria can be used to fetch missing messages from store nodes when network resumes.

By leveraging the store node to provide such query services, the application can mitigate the reliability issue of relay protocol.


## Security and Performance Considerations

The message query exposes the metadata of clients to the store nodes, and the store node can easily associate the messages with interested clents.
The query requests add a fair mount of load to store services, and increased linearly with more users onboarded. Store nodes should be able to scale up and scale down itself by monitoring or predicting the workload. 
The store node can also be a target for DDoS attack. The store node should have a mechanism to prevent such attack.
Application should provide user options to configure different store nodes, such nodes can either be self-hosted or public with better reputation.


## Implementation Suggestions (optional)

An optional *implementation suggestions* section may provide suggestions on how to approach implementation details, and, 
if available, point to existing implementations for reference.



## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

1. [Relay Protocol](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
