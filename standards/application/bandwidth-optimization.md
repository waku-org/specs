---
title: BANDWIDTH-OPTIMIZATION
name: Use Waku in a Bandwidth Efficient Way
category: Standards Track
tags: [reliability, application]
editor: Kaichao Sun <kaichao@status.im>
contributors:
---

## Abstract

Waku, as a real-time communication network, inherently uses network bandwidth for message routing. Its protocols are designed to facilitate the exchange of messages between peers in decentralized systems, but efficient bandwidth use is crucial to ensure scalability and resource management, especially for devices with limited connectivity or data plans.

Waku supports different protocols for routing messages, for example [Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) and [Lightpush](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) / [Filter](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md). Each protocol has its own strengths and trade-offs, making it essential to choose the right one based on use case and resource constraints.

This specification aims to provide guidance on how to use Waku in a bandwidth-efficient way, including best practices and potential optimizations for existing applications like [Status](https://status.app/).

## Best Practices

### Favors Lightpush/Filter Protocol on User Devices

**Relay protocol** propagates messages in real-time across the network but can result in significant bandwidth usage due to the broadcasting nature of message routing.

**Lightpush/Filter protocol** are more bandwidth-efficient as they minimize unnecessary data transmission, either by pushing messages to specific peers or filtering out irrelevant messages based on interested topics.

To optimize bandwidth usage on user devices, it is recommended to favor the Lightpush and Filter protocols when possible. The shortcoming of this approach is the application must provide access to service nodes to facilitate these protocols. Unlike the Relay protocol, which operates in a fully decentralized manner by broadcasting messages across the network, Lightpush and Filter require intermediary nodes to handle message forwarding and filtering.

In the long term, the Waku Network should implement an incentivization model to encourage the provision of Lightpush and Filter services, ensuring scalability, decentralization, and reliability. By incentivizing service node operators, the network can promote wider participation, distribute the operational load, and create a sustainable system for handling bandwidth-efficient protocols.

## Potential Issues and Optimizations

Multiple factors can cause excessive bandwidth usage, impacting overall bandwidth consumption in Status app. In following sections we will discuss some of the potential issues and optimizations.

### Global Shard Message Routing

Direct messages (DMs) and group chats are currently routed through the default shard /waku/2/rs/16/32, meaning every relay peer handles these messages. As user volume increases, this leads to exponential traffic growth, potentially overwhelming the network and making it unsustainable for users to bear the escalating bandwidth demands.

**Option 1**

A more practical approach would be to enforce the use of Lightpush and Filter protocols for global shard messages if such protocols are not enabled by default. This ensures that instead of relaying all messages across all peers, only relevant messages are pushed to nodes that have explicitly subscribed to them. This reduces overall network traffic and prevents unnecessary data from being routed to all peers, mitigating the risk of exponential traffic growth as the user base expands.

_Concerns:_
- depends on service nodes to facilitate Lightpush and Filter protocols, which may introduce centralization and privacy concerns.


**Option 2**

Implement a dynamic sharding system for global message routing. When a user joins the network, they are assigned a shard, and this shard index is broadcast to all their contacts. Additionally, shard information is embedded in contact links and contact request messages. Users can switch to a lower-traffic shard as needed, and any shard changes are broadcast to all their contacts to maintain communication consistency.

_Concerns:_
- handle the complexity when shard changes, e.g., when a user switches to a different shard, the application must ensure that all their contacts are aware of the change to avoid message loss.
- traffic increase could be exponential when adding more contacts. 

**Option 3**

Not joining the global shard relay network, instead fetching the messages from Store node periodically. 

_Concerns:_
- it will introduce additional latency and create a dependency on the availability and performance of the Store node, which result in bad UX.

### Message Retransmission in MVDS

Messages are resent when the recipient is offline, leading to additional network load.

Replace MVDS with a more reliable and bandwidth-efficient end-to-end (E2E) reliability protocol to reduce redundant message retransmissions.

### Updates for Descriptive Messages

Repeated publication of large messages, such as community descriptions, can result in frequent network consumption (e.g., every hour). Such messages include,
- Community descriptions published every hour.
- User profiles (TODO)

In the long term, such messages should persisted and updated with a decentralized storage provider like Codex.

In the short term, the application should optimize the frequency of updates and reduce the size of descriptive messages to minimize bandwidth usage, for example only publish the id or hash of the message content, store the original content in other places like IPFS or S3.

### Store Node Queries

Regular queries to store nodes for missing messages can create additional network traffic.

Use e2e reliability for missing messages retrieval.

### Device Synchronization

Synchronizing messages across multiple devices can increase bandwidth consumption.

Utilize a transient shard for syncing messages between devices to ensure efficient bandwidth use without relying on permanent storage.

### Backup of User Data

Frequent backups of user data, such as profile information, contacts, and communities (e.g., via ApplicationMetadataMessage_BACKUP), can be bandwidth-intensive.
See: [PR 2413](https://github.com/status-im/status-go/pull/2413) and [PR 2976](https://github.com/status-im/status-go/pull/2976).

The backup should be disabled by default, and manually triggered by the user when necessary. The application can also provide an option to schedule backups at specific intervals to reduce network traffic.

### Status Update Messages

Users might receive status update messages for themselves, which should be filtered out to avoid unnecessary bandwidth usage.

## Summary

Managing network bandwidth efficiently remains a critical challenge for Waku, especially as the network scales and more devices become connected, necessitating ongoing optimization of its protocols and architecture.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

1. [Relay Protocol](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
2. [Light Push Protocl](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)
3. [Filter Protocl](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md)
4. [MVDS - Minimum Viable Data Synchronization](https://github.com/vacp2p/rfc-index/blob/main/vac/2/mvds.md)
5. [End-to-end reliability for scalable distributed logs](https://forum.vac.dev/t/end-to-end-reliability-for-scalable-distributed-logs/293)
