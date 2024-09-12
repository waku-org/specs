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

Several scenarios can lead to excessive bandwidth usage:

**Global Shard Message Routing**

Direct messages (DMs) and group chats are routed through a global shard, causing higher network traffic.

**Message Retransmission in MVDS**

Messages are resent when the recipient is offline, leading to additional network load.

Replace MVDS with a more reliable and bandwidth-efficient end-to-end (E2E) reliability protocol to reduce redundant message retransmissions.

**Updates for Descriptive Messages**

Repeated publication of large messages, such as community descriptions, can result in frequent network consumption (e.g., every hour). Such messages include,
- Community descriptions published every hour.
- User profiles (TODO)

In the long term, such messages should persisted and updated with a decentralized storage provider like Codex.

In the short term, the application should optimize the frequency of updates and reduce the size of descriptive messages to minimize bandwidth usage, for example only publish the id or hash of the message content, store the original content in other places like IPFS or S3.

**Store Node Queries**

Regular queries to store nodes for missing messages can create additional network traffic.

Use e2e reliability for missing messages retrieval.

**Device Synchronization**

Synchronizing messages across multiple devices can increase bandwidth consumption.

Utilize a transient shard for syncing messages between devices to ensure efficient bandwidth use without relying on permanent storage.

**Backup of User Data**

Frequent backups of user data, such as profile information, contacts, and communities (e.g., via ApplicationMetadataMessage_BACKUP), can be bandwidth-intensive.
See: [PR 2413](https://github.com/status-im/status-go/pull/2413) and [PR 2976](https://github.com/status-im/status-go/pull/2976).

The backup should be disabled by default, and manually triggered by the user when necessary. The application can also provide an option to schedule backups at specific intervals to reduce network traffic.

**Status Update Messages**

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
