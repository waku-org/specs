---
title: TRANSPORT-RELIABILITY
name: Waku Transport Reliability 
category: Standards Track
tags: [reliability, application]
editor: Kaichao Sun <kaichao@status.im>
contributors:
  - Richard Ramos <richard@status.im>
---

## Abstract

Waku provides an efficient transport layer for p2p communications.
It defines protocols like [Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) and [Lightpush](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) / [Filter](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md) for routing messages in decentralised networks.
However, there is no guarantee that a message broadcast in a Waku network will reach its destination.

For example, the receiver in a chat application using Waku as p2p transport may miss messages when a network issue happens at either the sender or the receiver side. 

In general, a message in a Waku network may be in one of 3 states from the sender's perspective:

- **outgoing**, the message is posted by the sender but no confirmations from other nodes yet
- **sent**, the message is received by any other node in the network
- **delivered**, the message is acknowledged on the application layer by the intended recipient

Application like Status already uses [MVDS](https://github.com/vacp2p/rfc-index/blob/main/vac/2/mvds.md) for e2e acknowledgement in direct messages and group chat. Also there is an ongoing [discussion](https://forum.vac.dev/t/end-to-end-reliability-for-scalable-distributed-logs/293) about a more general and bandwidth efficient solution for e2e reliablity.

In other words, an application defines a payload over Waku and is interested in e2e delivery between application users. Waku provides a pub/sub broadcast transport, which is interested in reliably routing a message to all participants in the broadcast group.

Before we have a complete design for e2e reliability, we need to compose existing protocols to increase the reliability of the transport protocol. This document proposes a few options for such composition. 

## Motivation

The [Store protocol](https://github.com/waku-org/specs/blob/master/standards/core/store.md) provides a way for nodes in the network to query the existence of messages or fetch specific messages based on the search criteria.

**Query criteria with message hash**

For the nodes that may have connection issues to **publish** messages via transport layer, this search criteria can be used to check whether a message is populated in the network or not. The message exists in store node can be marked from `outgoing` to `sent` by application. If the message is not found in the store node, the application can resend the message.

**Search criteria with topics and time range**

For the nodes that may have connection issues to **receive** messages via transport layer, this search criteria can be used to fetch missing messages from store nodes periodically after network resumes. 

In summary, by leveraging the store node to provide such query services, the applications are able to mitigate reliability issues on the Waku transport layer.
This approach also introduces new limitations like centralized points of failures in Store nodes, diminished privacy, and lower scalability.
It should be viewed as a temporary solution and deprecated when an e2e reliability solution is ready.

## Implementation Suggestions 

### Query with Message Hash

For outgoing messages, the processing flow can be like this:
- create a buffer for all "outgoing" message hashes
- send message via relay or lightpush protocol
- add message hash to the buffer
- keep a copy of the message locally with status outgoing
- check the buffer periodically
- query the store node with message hash in the buffer of which the send attempt was more than a few seconds ago
- if the message exists, update its status to "sent" in local data store and remove the message hash from the buffer
- if the message does not exist, resend the message
- if the message is still missing in the store node for a period of time, trigger the message failed to send workflow and remove the message hash from the buffer

The implementation in Python may look like this:

```python
outgoingMessageHashes = []

class Message:
    hash: str
    postTime: int
    status: str
    content: str

def send(message):
    # send message via relay or lightpush protocol, here use relay as example
    waku.relay.post(message)
    outgoingMessageHashes.append(message.hash)

    message.status = 'outgoing'
    database.saveMessage(message)

def checkOutgoingMessages(peerID):
    for messageHash in outgoingMessageHashes:
        message = database.getMessage(messageHash)
        # only query store node for ongoing message, and posted more than 3 seconds ago
        if message.status == 'ongoing' && time.now() - message.postTime > 3:
            response = waku.store.queryMessage(peerID, messageHash)
            if response.exists():
                database.updateMessageStatus(messageHash, 'sent')
                outgoingMessageHashes.remove(messageHash)
            elif time.now() - message.postTime > 10:
                # resend the message if it's not stored in store node after 10 seconds
                waku.relay.post(message)
```

Function `checkOutgoingMessages` is called periodically, most likely every a few seconds. Message hashes can be queried in batch to reduce the number of requests to store nodes, the size in a batch shoud not exceed the max supported size by store node.

The store node can be set and updated directly by application or selected from peers which are discovery by protocols like [discv5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md) or [peer exchange](https://github.com/waku-org/specs/blob/master/standards/core/peer-exchange.md).

The store node may only support specific pubsub topics, and the application should group message hashes by pubsub topic before sending the request.

When persistent network issue happens, you may not want to resend the failed messages indefinitely, the application should have a mechanism to clean the cache with failed message hashes and trigger other retry logic after a few attempts.

### Query with Topics and Time Range

An application could use different pubsub topics and content topics, for example a community may have its own pubsub topic, and each channel may have its own content topic. To fetch all missing messages in a specific channel, the application can query the store node with the provided pubsub topic, content topic and time range.

For incoming messages, the processing flow can be like this:
- subscribe to the interested pubsub and content topics
- query the store node with the interested topics and time range for message hashes periodically
- check if each received message hash already exists in the local database. if not, add the missing message hash to a buffer.
- batch fetch the full messages corresponding to the missing message hashes in the buffer from the store node
- process the messages
- update the last fetch time for the interested topic

The implementation in Python may look like this:

```python
class FetchRecord:
    pubsubTopic: str
    lastFetch: int

class QueryParams:
    pubsubTopic: str
    contentTopics: List[str]
    fromTime: int
    toTime: int

def fetchMissingMessages(peerID, queryParams):
    missingMessageHashes = []

    # get missing message identifiers first in order to reduce the data transfer
    response = waku.store.queryMessageHashes(peerID, queryParams)
    for !response.isComplete():
        # process each message in the response
        response.messages().forEach(messageHash -> {
            message = queryDbMessageByHash(messageHash)
            if message.exists():
                continue
            }
            missingMessageHashes.append(messageHash)
        })

        # process next page of the response
        response.Next()
    
    # fetch missing messages with hashes in batch
    response = waku.store.queryMessagesByHash(peerID, missingMessageHashes)
    response.messages().forEach(message -> {
        processMessage(message)
    })

    updateFetchRecord(queryParams.pubsubTopic, queryParams.toTime)
```

`QueryParam` includes all the necessary information to fetch missing messages. The application should iterate all the interested pubsub topics, along with its content topics to construct the `QueryParam`.

Function `fetchMissingMessages` is runing periocally, for example 1 minute. It first fetch all the message hashes in the specified time range, check if message exist in local dabatase, if not, fetch the missing messages in batch. The batch size should be bounded to avoid large data transfer or exceed the max supported size by store node. 

When finishing fetching missing messages, the application should update the last fetch time in `FetchRecord`. The last fetch time can be used to calculate the time range for the next fetch and avoid fetching the same messages again.


### Unified Query

There are cases that both outgoing and incoming messages are queried in similar situation, for example at same interval. The application can combine the above two worflows into one to have a unified query with better performance overall.

The workflow can be like this:
- create outgoing buffer for all "outgoing" messages
- create incoming buffer for all recently received message hashes
- periodically query store node based on interested topics and time range for message hashes
- check outgoing buffer with returned message hash, if included, mark message as `sent`, resend if needed
- check incoming buffer with returned message hash, if not included, fetch the missing message with its hash

## Security and Performance Considerations

The message query request exposes the metadata of clients to the store nodes, and the store node is capable to associate the messages with interested clents.

The query requests add a fair amount of load to store nodes, and increased linearly with more users onboarded. Store nodes should be able to scale up and scale down itself by monitoring or predicting the workload. 

The store node can also be a target for DDoS attack. The store node should have a mechanism to prevent such attack.

Application should provide options to configure different store nodes for its users, such nodes can either be self-hosted or public with better reputation.


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

1. [Relay Protocol](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
2. [Light Push Protocl](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)
3. [Filter Protocl](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md)
4. [MVDS - Minimum Viable Data Synchronization](https://github.com/vacp2p/rfc-index/blob/main/vac/2/mvds.md)
5. [End-to-end reliability for scalable distributed logs](https://forum.vac.dev/t/end-to-end-reliability-for-scalable-distributed-logs/293)
6. [Waku Store Query](https://github.com/waku-org/specs/blob/master/standards/core/store.md)
7. [Waku v2 Discv5 Ambient Peer Discovery](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md)
8. [Waku v2 Peer Exchange](https://github.com/waku-org/specs/blob/master/standards/core/peer-exchange.md)
