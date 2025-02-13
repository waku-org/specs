---
title: WAKU-MIX
name: Waku Mix
editor: Prem Chaitanya Prathi <premprathi@proton.me>
contributors:
 - Akshaya Mani <akshaya@status.im>
 - Hanno Cornelius <hanno@status.im>
---

## Tags

`waku/core-protocol` 

# Abstract
The document describes [libp2p mix](https://rfc.vac.dev/vac/raw/mix/) integration into waku. 
This would provide higher anonymity for users publishing or querying for messages to/from the Waku network. 

This document covers integration of mix with [lightpush](https://rfc.vac.dev/waku/standards/core/19/lightpush) and [store](https://rfc.vac.dev/waku/standards/core/13/store) protocols. 
It also covers the aspect of relay nodes acting as mix nodes.
Both of these are request response based protocols that follow a service-user and service-provider model.

Integration of gossipsub with mix is not covered in this document.

## Background / Rationale / Motivation

Waku protocols have weak sender/originator anonymity as explained in [Waku Privacy and Anonymity Analysis](https://vac.dev/rlog/wakuv2-relay-anon/).
Without further anonymization, it is easy to analyze the network traffic and determine originator of messages published into the network or determine the topic interests by analyzing store queries requested by a user. 
Similar analysis is possible for topic subscriptions via [Filter](https://rfc.vac.dev/waku/standards/core/12/filter) protocol as well.

By using the libp2p mix protocol, we can improve the anonymity for publishers and store query users in the waku network.
Anonymity of Filter users is not be addressed by this document and will have to be addressed separately.

## Theory / Semantics

Waku Mix spans an overlay network of all the waku nodes that support the mix protocol. 

Nodes with `mix` protocol mounted SHOULD advertise that they support mix protocol via their their chosen discovery method.
They may do so by updating their [ENR](#enr-updates) and using one of the ENR based discovery methods.

Nodes that would like to use `mix` SHOULD discover enough mix nodes so that there is always a healthy pool of mix nodes available for selection.
The pool size of mix nodes SHOULD be larger then `100` for the mixing to be effective.

Nodes that want higher anonymity while publishing a message or performing a store query SHOULD use the `mix` protocol to route their messages to the destination.

In order to preserve the originator identify from the service node, the reply for a request SHOULD be implemented via `Single Use reply blocks`.

A node that sends message using `mix` MAY use 2 redundant mix paths to have better reliability of the message being delivered.

## Terminology
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## Node Roles

Mix protocol defines 3 roles for the nodes in the mix network - `entry`, `exit`, `intermediary`. 
Each node in the network can have all 3 roles or can act as just an entry/exit node as well. 

- An `entry` node is the originator node of a message, i.e a node that wishes to publish/query messages to/from the waku network.
- An `exit` node is responsible for delivery messages to destination peer in the network.
- An `intermediary` node is responsible for just forwarding mix packet to the next mix node in the path.

All the relay nodes would by default act as mix `intermediary` and `exit` nodes in the network. 
Any node that wishes to publish/query messages from the waku network would act as an entry node. 
The implementation can provide a configuration to disable a node from acting as an `intermediary` node.

Light/Edge nodes will only be acting as `entry` nodes.

## Flow

TBD

## ENR updates

Each waku node that supports `mix` SHOULD indicate this in its discoverable [ENR](https://github.com/waku-org/specs/blob/master/standards/core/enr.md).
The following fields MUST be set as part of its discoverable ENR:
- The `bit 5` in the [waku2 ENR key](https://github.com/waku-org/specs/blob/master/standards/core/enr.md#waku2-enr-key) is reserved to indicate `mix`support.This bit must be set to true to indicate `mix` support.
- a new field `mix-key` set to the `ed25519 public key`which is used for sphinx encryption.

### Discovery

Mix protocol provides better anonymity when an originator node has a large pool of mix nodes to choose randomly from. 
This moves the problem into discovery domain and requires the following from discovery mechanisms:
1. it is important for nodes to be able to discover as many nodes as possible quickly. This becomes especially important for edge nodes that come online just to publish/query messages for a short period of time.
2. it is important to have the most recent online status of the nodes so that broken mix paths are not selected leading to reliability issues.

Point-2 above can be mitigated partially by choosing redundant mix paths for the same message by the originator but this may not be effective.

## Spam Protection

Mix protocol in waku network SHOULD require `rate-limiting/spam` protection to handle scenarios such as:

1. any node can generate a mix packet and publish into the mix network. Hence there needs to be some validation as to who is allowed to publish and whether the user is within allowed rate-limits.
2. any node can generate paths which are broken intentionally and send messages into the mix network.
3. an attacker can spawn a huge number of mix nodes so that user behaviour is observed in order to determine traffic patterns and deanonymize users. 

The current [WAKU RLN](https://rfc.vac.dev/waku/standards/core/17/rln-relay) can address scenario-3 by making it restrictive for users to be able to spawn a lot of nodes in the network.

There is a need to enforce rate-limits and spam protect the mix network. 

## Tradeoffs

The following could be an impact of using a mixing network for publishing and querying messages. The impact needs to be analyzed to take further decisions as to whether using mix would be a default behaviour for using waku protocols such as relay etc.

- Each mix nodes adds a delay while forwarding the message to the next hop. With a path length of 3 and avg delay of 3 msecs and including packet processing and connection established delay between mix nodes, the overall delay could be significant. 
- Effectiveness of a mixing network depends on how many nodes in the network support mix and what is the set of nodes an originating node considers to select a random subset of nodes to form a path. ?????

# Implementation Suggestions

TBD

## Parameters 

TBD

## Attack Vectors
TBD

## Security/Privacy Considerations

A standard track RFC in `stable` status MUST feature this section.
A standard track RFC in `raw` or `draft` status SHOULD feature this section.
Informational RFCs (in any state) may feature this section.
If there are none, this section MUST explicitly state that fact.
This section MAY contain additional relevant information,
e.g. an explanation as to why there are no security consideration
for the respective document.

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References
 - [libp2p mix](https://rfc.vac.dev/vac/raw/mix/)
 - [waku lightpush](https://rfc.vac.dev/waku/standards/core/19/lightpush)
 - [waku relay](https://rfc.vac.dev/waku/standards/core/11/relay)
 - [Single Use Reply Blocks]()