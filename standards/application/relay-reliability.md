---
title: RELIABILITY-RELAY
name: Reliability for Relay Protocol
category: Standards Track
tags: [reliability, application]
editor: Kaichao Sun <kaichao@status.im>
contributors:
  - Richard Ramos <richard@status.im>
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

## Implementation Suggestions 

TODO

### Query with Message Hash

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

def checkOutgoingMessages():
    for messageHash in outgoingMessageHashes:
        message = database.getMessage(messageHash)
        # only query store node for ongoing message, and posted more than 3 seconds ago
        if message.status == 'ongoing' && time.now() - message.postTime > 3:
            response = waku.store.queryMessage(messageHash)
            if response.exists():
                database.updateMessageStatus(messageHash, 'sent')
                outgoingMessageHashes.remove(messageHash)
            elif time.now() - message.postTime > 10:
                # resend the message if it's not stored in store node after 10 seconds
                waku.relay.post(message)
```

Function `checkOutgoingMessages` is called periodically. Message hashes can be queried in batch to reduce the number of requests to store nodes, the size in a batch shoud not exceed the max supported size by store node.

The store node can be set and updated directly by application or selected from peer discovery protocol like [discv5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md) or [peer exchange](https://github.com/waku-org/specs/blob/master/standards/core/peer-exchange.md).

The store node may only support specific pubsub topics, and the application should group message hashes by pubsub topics before querying the store node. 

When persistent network issue happens, you may not want to resend the failed messages indefinitely, the application should have a mechanism to clean the cache with failed message hashes and trigger other retry logic after a few attempts.

### Query with Topics and Time Range

```python

## Security and Performance Considerations

The message query request exposes the metadata of clients to the store nodes, and the store node can easily associate the messages with interested clents.

The query requests add a fair mount of load to store services, and increased linearly with more users onboarded. Store nodes should be able to scale up and scale down itself by monitoring or predicting the workload. 

The store node can also be a target for DDoS attack. The store node should have a mechanism to prevent such attack.

Application should provide options to configure different store nodes for its users, such nodes can either be self-hosted or public with better reputation.


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

1. [Relay Protocol](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)