---
title: RELIABILITY-DAG
name: Message References with DAG
tags: waku/application
editor: Kaichao <kaichao@status.im>
contributors: 
---

## Abstrct

Waku provides a decentralized [relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) network for routing messages. It's not possible to garantee a meesage is reached to its receivers without introduce any centralized coordinator. [MVDS](https://github.com/vacp2p/rfc-index/blob/e5b859abfb3e42fde4336e5fc5b4e7250126f8ce/vac/2/mvds.md) is a way to ensure message delivery by sending acks by recipents frequently. It's not efficient in low-bandwith environments like IoT devices or mobile phones. If the group or community becomes big enough, the cost of bandwith also increase dramatically.

The present documentation proposes a way to reference messages from group members when sending a new message. In this way, the messages of the group compose a Directed Acyclic Graph (aka. DAG) effectively. By looking into the tips of the graph, the member in the group can get the missing messages from [store]() node with the refrence specified in the tip message. Each member also send a checkpoint message periodically.

## Terms 

**Reference of a message (reference id)**
The hash of the message define in [Deterministic Message Hashing](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md#deterministic-message-hashing). However, current implementation of message hash can not be used reliably in scenarios like message retransimission. The realy protocol expects a "recent" timestamp in the `WakuMessage`, retransmission will failed if passed the time window. Either we introduce payload hash as the reference id or modify the calculation of message hash to exclude timestamp. In the first option, the payload hash based query of message to store node is requried to fulfill our requirements.

**Tip message**
The latest message from a member. It can choose to reference or not reference the sender's previous message. It must reference the last seen messages from other `n` members in the group.

**Control message**
The message which is not input by user, but create and send automatically in application to facilate our design.

**Checkpoint message**
It's a type of control message from a group member, which only contains references. Such message should not be persistent in any nodes, instead it's only used to find out if the referenced message is missing or not by other group members.

**Request message**
It's another type of control message for asking a message by its reference id. It can be persistent in store and sender node but not others. Group members will only fulfill the request message if store node doesn't have it. The sender node regularly check if pending request message gets fulfilled or not.

## Message Flows

**Sending and retrieving messages**
- Alice creates a group `G`
- Bob, Charlie, Dave join `G`.
- Alice sends message `a1` in `G` without any references.
- Bob sends `b1` in `G` with reference to `a1`.
- Charlies send `c1` with reference to `a1` and `b1`.
- Dave sends `d1` with reference to `b1`, and `c1`.
- Alice sends `a2` with reference to `a1`, `b1`, and `d1`.
- Charlie receives `a2`, finds out that `d1` is not exist in local, he wants to retrieve the message from store node by the reference id.
- If the `d1` did not exist in store node, charlie broadcasts a `request` message in the group.
- Dave gets the `request` message and check if `d1` exists in local or not. 
- Dave has `d1` and broardcast it to the network.


## Payload Format

The references are generally encrypted in `WakuMessage.payload`. 

```python
class Payload:
    def __init__(self, references):
        # omit other attributes
        self.references = references
```

For test purpose, it can also be encoded in `WakuMessage.meta` for public analysis in following manner,

```python
concat(message_hash_1, message_hash_2, message_hash_3)
```

## Principle

### Choose Reference

The algorithm to choose references is basically randomly but give active users higher probability. Online users are generally seen as active. If app doesn't have online/offline capability, it's acceptable to say the user who has sent out any messages in the last `n` (for example, 7) days is treated as active. Below is a recommend implementation in Python, 

```python
import random

def weighted_choose(group1, group2, prob_group1, k):
    combined_group = group1 + group2
    weights = [prob_group1 / len(group1)] * len(group1) + [(1 - prob_group1) / len(group2)] * len(group2)

    # Ensure k does not exceed the number of unique elements
    k = min(k, len(set(combined_group)))

    # Generate a weighted choice list ensuring no duplicates
    selected = []
    while len(selected) < k:
        choice = random.choices(combined_group, weights, k=1)[0]
        if choice not in selected:
            selected.append(choice)
    
    return selected

# Example usage
active_group = ['Alice', 'Bob', 'Charlie']
inactive_group = ['Diana', 'Eve', 'Frank']
prob_active = 0.7  # 70% chance of picking a member from active group
k = 4  # Number of unique members to pick

picked_members = weighted_choose(active_group, inactive_group, prob_active, k)

print("Picked members:", picked_members)
```

**Note**: It's recommended to always include a reference to your previous message, and two or three references to other group members. If the group is small (less than 5 members), one reference to other members is acceptable. If the group is large and the length of message payload is not a limitation, following curve can be considered,

```
f = a * log(x)
```


### Checkpoint Frequency

To protect the group from too many checkpoint messages, the interval to broadcast checkpoint message for each member is,

`p = f * n`

- `f`: the expected interval for all checkpoint message, for example 3 seconds.
- `n`: the count of online members, if no such status, `n` can be estimated with a certain percentage (for example 10%) of total group members.

A new checkpoint message should be send out immediately after user starts the application, followed by periodically checkpoints.


### Recursive Retrieve

If the user comes online after a long period, it's recommended to use store query for catch up.
The depth of recursive retrieve can be limited up to a certain number (for example 5) for performance.


## Security Consideration

*TODO*


## References

1. [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
2. [14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md)
3. [2/MVDS](https://github.com/vacp2p/rfc-index/blob/main/vac/2/mvds.md)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
