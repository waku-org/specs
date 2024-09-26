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

Multiple factors can cause excessive bandwidth usage, impacting overall bandwidth consumption in Status app. In following sections we will discuss some of the potential issues and options for optimizations, different options can be utilized together to achieve the best result.

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

Direct messages, group messages and some community join request messages are sent via MVDS. When the recipient is offline, these messages are repeatedly resent until a predefined limit is reached, which leads to more bandwidth usage.

**Option 1**

Replace MVDS with a more reliable and bandwidth-efficient end-to-end (E2E) [reliability protocol](https://forum.vac.dev/t/end-to-end-reliability-for-scalable-distributed-logs/293) to reduce message retransmissions.

_Concerns:_
- fully adopting e2e reliability protocol requires significant time and resources for implementation.

**Option 2**

Disable MVDS retransmission for recipients that are not online/active.

_Concerns:_
- user may set their status to offline but still want to receive messages. (TODO needs input from Status team)

**Option 3**

Increase the time interval between message retransmissions to reduce bandwidth usage. The current resend epoch calculation in Status app is as follows:

```
next_epoch = current_epoch + (2^(send_count−1)×30×3) + rand(0,30)
```

The interval can be increased by adjusting the constant factor (30) to a higher value, e.g., 60 or 90, to reduce the frequency of message retransmissions.

_Concerns:_
- it may increase the latency of message delivery.


### Community Description

Refer to [Optimizing Community Description](https://forum.vac.dev/t/optimizing-community-description/339)

### Store Node Queries for missing messages and messages sent check

Regular queries to store nodes can create additional network traffic.

**Option 1**

Use e2e reliability for missing messages retrieval and messages sent check.


### Device Synchronization

 To ensure a consistent user experience across multiple devices, a lot of messages are sent through Waku global shard, for example user profile, contacts, communities, activies, etc.

[Scope of user profile data: what we need to backup to waku and sync](https://docs.google.com/spreadsheets/d/1VyVIg5ZaVSlKWZRHSLzSMCMGJsbprkWUfEqCUJKma8w/edit?usp=sharing)

**Option 1**

Change the product design to only allow sync through a negotiated shard when two devices are online at the same time.
Waku can allocate a range of shards specifically for syncing messages, allowing users to subscribe to a shard only when needed. From the user's perspective, this operates as a transient shard, dynamically utilized for short-term tasks.


_Concerns:_
- UX is not matched with the current design.

**Option 2**

Do not sync messages through Waku, instead use a bittorrent or other systems to sync history messages, and use the following backup flow for profile, contacts, communities synchronization.

### User Data Backup

Frequent backups of user data, such as profile information, contacts, and communities ((e.g., `ApplicationMetadataMessage_BACKUP`)), can be bandwidth-intensive. It's also tightly coupled with the device synchronization.

See: [PR 2413](https://github.com/status-im/status-go/pull/2413) and [PR 2976](https://github.com/status-im/status-go/pull/2976).

**Option 1**

Waku provides a new protocol (namely, stateful store),
- there needs to be another table in store node's database, let’s call it states for now
- each record in states table has fields (id, pubkeys, content, pubsubTopic, contentTopic), the pubkeys is a list of public keys of user's devices.
- we will assign a new shard to route the messages of state creation and updates.

How user backup their data, 
- send user profile along with the device's pubkey and signature of the message hash
- store node receives the message, verifies the content in message with the bundled pubkey, further save it to the state table. (TODO Status app may use the same key for different devices, need to confirm)
- when new update event happens, it sends a new message just like the previous one
- store node further check and updates the content

_Note:_

To handle the conflict between different devices, we use LWW (Last Write Wins) strategy, which means the last update message will be saved to the state table. 
Each devices should fetch the latest stete from store, update local state, then compose and send the new backup message to store.

_Concerns:_

- it requires a significant amount of work to implement the stateful store protocol.

**Option 2**

The backup should be disabled by default, and manually triggered by the user when necessary. 
User can enable periodically backup and set the intervals as needed. Currently, the periodically backup is enabled by default.

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
6. [Optimizing Community Description](https://forum.vac.dev/t/optimizing-community-description/339)
