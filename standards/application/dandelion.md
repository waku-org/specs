---
title: DANDELION
name: Waku v2 Dandelion
category: Standards Track
tags: [waku/anonymity]
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

## Abstract

This document specifies a deanonymization mitigation technique,
based on [Dandelion](https://arxiv.org/abs/1701.04439) and [Dandelion++](https://arxiv.org/abs/1805.11060),
for Waku Relay.
It mitigates mass deanonymization in the [multi-node (botnet) attacker model](../../informational/adversarial-models.md/#multi-node),
even when the number of malicious nodes is linear in the number of total nodes in the network.

Based on the insight that symmetric message propagation makes deanonymization easier,
it introduces a probability for nodes to simply forward the message to one select relay node
instead of disseminating messages as per usual relay operation.

## Background and Motivation

[Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md), offers privacy, pseudonymity, and a first layer of anonymity protection by design.
Being a modular protocol family [Waku v2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)
offers features that inherently carry trade-offs as separate building blocks.
Anonymity protection is such a feature.
The [Anonymity Trilemma](https://freedom.cs.purdue.edu/projects/trilemma.html)
states that an anonymous communication network can only have two out of
low bandwidth consumption, low latency, and strong anonymity.
Even when choosing low bandwidth and low latency, which is the trade-off that basic Waku Relay takes,
better anonymity properties (even though not strong per definition) can be achieved by sacrificing some of the efficiency properties.
WAKU2-DANDELION specifies one such technique, and aims at gaining the best "bang for the buck"
in terms of efficiency paid for anonymity gain.
WAKU2-DANDELION is based on [Dandelion](https://arxiv.org/abs/1701.04439)
and [Dandelion++](https://arxiv.org/abs/1805.11060).

Dandelion is a message spreading method, which, compared to other methods,
increases the uncertainty of an attacker when trying to link messages to senders.
Libp2p gossipsub aims at spanning a [d-regular graph](https://en.wikipedia.org/wiki/Regular_graph) topology, with d=6 as the [default value](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/29/config.md/#gossipsub-v10-parameters).
Messages are forwarded within this (expected) symmetric topology,
which reduces uncertainty when trying to link messages to senders.
Dandelion breaks this symmetry by subdividing message spreading into a "stem" and a "fluff" phase.

In the "stem" phase, the message is sent to a single select relay node.
With a certain probability, this message is relayed further on the "stem",
or enters the fluff phase.
On the stem, messages are relayed to single peers, respectively,
while in fluff phase, messages are spread as per usual relay operation (optionally augmented by random delays to further reduce symmetry).
The graph spanned by stem connections is referred to as the anonymity graph.

Note: This is an early raw version of the specification.
It does not strictly follow the formally evaluated Dandelion++ paper,
as we want to experiment with relaxing (and strengthening) certain model properties.
In specific, we aim at a version that has tighter latency bounds.
[Further research](https://arxiv.org/pdf/2201.11860.pdf) suggests that Dandelion++'s design choices are not optimal,
which further assures that tweaking design choices makes sense.
We will refine design decisions in future versions of this specification.

Further information on Waku anonymity may be found in our [Waku Privacy and Anonymity Analysis](https://vac.dev/wakuv2-relay-anon).

## Theory and Functioning

WAKU2-DANDELION can be seen as an anonymity enhancing add-on to [Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md) message dissemination,
which is based on [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md).
WAKU2-DANDELION subdivides message dissemination into a "stem" and a "fluff" phase.
This specification is mainly concerned with specifying the stem phase.
The fluff phase corresponds to [Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md),
with optional fluff phase augmentations such as random delays.
Adding random delay in the fluff phase further reduces symmetry in dissemination patterns and
introduces more uncertainty for the attacker.
Specifying fluff phase augmentations is out of scope for this document.

_Note:
We plan to add a separate specification for fluff phase augmentations.
We envision stem and fluff phase as abstract concepts.
The Dandelion stem and fluff phases instantiate these concepts.
Future stem specifications might comprise: none (standard relay), Dandelion stem, Tor, and mix-net.
As for future fluff specifications: none (standard relay), diffusion (random delays), and mix-net._

Messages relayed by nodes supporting WAKU2-DANDELION are either in stem phase or in fluff phase.
We refer to the former as a stem message and to the latter as a fluff message.
A message starts in stem phase, and at some point, transitions to fluff phase.
Nodes, on the other hand, are in stem state or fluff state.
Nodes in stem state relay stem messages to a single relay node, randomly selected per epoch for each incoming stem connection.
Nodes in fluff state transition stem messages into fluff phase and relay them accordingly.
Fluff messages are always disseminated via Waku Relay,
by both nodes in stem state and nodes in fluff state.

Messages originated in the node (i.e. messages coming from the application layer of our node),
are always sent as stem messages.

The stem phase can be seen as a different protocol, and messages are introduced into Waku Relay, and by extension gossipsub,
once they arrive at a node in fluff state for the first time.
WAKU2-DANDELION uses [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) as the protocol for relaying stem messages.

There are no negative effects on gossipsub peer scoring,
because Dandelion nodes in _stem state_ still normally relay Waku Relay (gossipsub) messages.

## Specification

Nodes $v$ supporting WAKU2-DANDELION MUST either be in stem state or in fluff state.
This does not include relaying messages originated in $v$, for which $v$ SHOULD always be in stem state.

### Choosing the State

On startup and when a new epoch starts,
node $v$ randomly selects a number $r$ between 0 and 1.
If $r < q$, for $q = 0.2$, the node enters fluff state, otherwise, it enters stem state.

New epochs start when `unixtime` (in seconds) $\equiv 0 \mod 600$,
corresponding to 10 minute epochs.

### Stem State

On entering stem state,
nodes supporting WAKU2-DANDELION MUST randomly select two nodes for each pubsub topic from the respective gossipsub mesh node set.
These nodes are referred to as stem relays.
Stem relays MUST support [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md).
If a chosen peer does not support [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md),
the node SHOULD switch to fluff state.
(We may update this strategy in future versions of this document.)

Further, the node establishes a map that maps each incoming stem connection
to one of its stem relays chosen at random (but fixed per epoch).
Incoming stem connections are identified by the [Peer IDs](https://docs.libp2p.io/concepts/peers/#peer-id/)
of peers the node receives [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) messages from.
Incoming [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) connections from peers that do not support WAKU2-DANDELION are identified and mapped in the same way.
This makes the protocol simpler, increases the anonymity set, and offers Dandelion anonymity properties to such peers, too.

The node itself is mapped in the same way, so that all messages originated by the node are relayed via a per-epoch-fixed Dandelion relay, too.

While in stem state, nodes MUST relay stem messages to the respective stem relay.
Received fluff messages MUST be relayed as specified in the fluff state section.

The stem protocol ([19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)) is independent of the fluff protocol ([Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)).
While in stem state, nodes MUST NOT gossip about stem messages,
and MUST NOT send control messages related to stem messages.
(An existing gossipsub implementation does _not_ have to be adjusted to not send gossip about stem messages,
because these messages are only handed to gossipsub once they enter fluff phase.)

#### Fail Safe

Nodes $v$ in stem state SHOULD store messages attached with a random timer between $t_1 = 5 * 100ms$ and $t_2 = 2 * t_1$.
This time interval is chosen because

- we assume $100\,ms$ as an average per hop delay, and
- using $q=0.2$ will lead to an expected number of 5 stem hops per message.

If $v$ does not receive a given message via Waku Relay (fluff) before the respective timer runs out,
$v$ will disseminate the message via Waku Relay.

### Fluff State

In fluff state, nodes operate as usual Waku Relay nodes.
The Waku Relay functionality might be augmented by a future specification, e.g. adding random delays.

Note: The [Dandelion](https://arxiv.org/abs/1701.04439) paper describes the fluff phase as regular forwarding.
Since Dandelion is designed as an update to the Bitcoin network using diffusion spreading,
this regular forwarding already comprises random delays.

## Implementation Notes

Handling of the WAKU2-DANDELION stem phase can be implemented as an extension to an existing [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) implementation.

Fluff phase augmentations might alter gossipsub message dissemination (e.g. adding random delays).
If this is the case, they have to be implemented on the libp2p gossipsub layer.

## Security/Privacy Considerations

### Denial of Service: Black Hole Attack

In a [black hole attack](../../informational/adversarial-models.md/#black-hole-internal), malicious nodes prevent messages from being spread,
metaphorically not allowing messages to leave once they entered.
This requires the attacker to control nodes on all dissemination paths.
Since the number of dissemination paths is significantly reduced in the stem phase,
Dandelion spreading reduces the requirements for a black hole attack.

The fail-safe mechanism specified in this document (proposed in the Dandelion paper), mitigates this.

### Anonymity Considerations

#### Attacker Model and Anonymity Goals

WAKU2-DANDELION provides significant mitigation against mass deanonymization in the
passive [scaling multi node model](../../informational/adversarial-models.md/#scaling-multi-node).
in which the attacker controls a certain percentage of nodes in the network.
WAKU2-DANDELION provides significant mitigation against mass deanonymization
even if the attacker knows the network topology, i.e. the anonymity graph and the relay mesh graph.

Mitigation in stronger models, including the _active scaling multi-node_ model, is weak.
We will elaborate on this in future versions of this document.

WAKU2-DANDELION does not protect against targeted deanonymization attacks.

#### Non-Dandelion Peers

Stem relays receiving messages can either be in stem state or in fluff state themselves.
They might also not support WAKU2-DANDELION,
and interpret the message as classical [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md),
which effectively makes them act as fluff state relays.
While such peers lower the overall anonymity properties,
the [Dandelion++ paper](https://arxiv.org/abs/1805.11060)
showed that including those peers yields more anonymity compared to excluding these peers.

### Future Analysis

The following discusses potential relaxations in favour of reduced latency,
as well as their impact on anonymity.
This is still work in progress and will be elaborated on in future versions of this document.

Generally, there are several design choices to be made for the stem phase of a Dandelion-based specification:

1. the probability of continuing the stem phase, which determines the expected stem lengh,
2. the out degree in the stem phase, which set to 1 in this document (also in the Dandelion papers),
3. the rate of re-selecting stem relays among all gossipsub mesh peers (for a given pubsub topic), and
4. the mapping of incoming connections to outgoing connections.

#### Bound Stem Length

Choosing $q = 0.2$, WAKU2-DANDELION has an expected stem length of 5 hops,
Assuming $100ms$ added delay per hop, the stem phase adds around 500ms delay on average.

There is a possibility for the stem to grow longer,
but some applications need tighter bounds on latency.

While fixing the stem length would yield tighter latency bounds,
it also reduces anonymity properties.
A fixed stem length requires the message to carry information about the remaining stem length.
This information reduces the uncertainty of attackers
when calculating the probability distribution assigning each node a probability for having sent a specific message.
We will quantify the resulting loss of anonymity in future versions of this document.

#### Stem Relay Selection

In its current version, WAKU2-DANDELION nodes default to fluff state
if the random stem relay selection yields at least one peer that does not support [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) (which is the stem protocol used in WAKU2-DANDELION.
If nodes would reselect peers until they find peers supporting [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md),
malicious nodes would get an advantage if a significant number of honest nodes would not support [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md).
Even though this causes messages to enter fluff phase earlier,
we choose the trade-off in favour of protocol stability and sacrifice a bit of anonymity.
(We will look into improving this in future versions of this document.)

#### Random Delay in Fluff Phase

[Dandelion](https://arxiv.org/abs/1701.04439) and [Dandelion++](https://arxiv.org/abs/1805.11060)
assume adding random delays in the fluff phase as they build on Bitcoin diffusion.
WAKU2-DANDELION (in its current state) allows for zero delay in the fluff phase and outsources fluff augmentations to dedicated specifications.
While this lowers anonymity properties, it allows making Dandelion an opt-in solution in a given network.
Nodes that do not want to use Dandelion do not experience any latency increase.
We will quantify and analyse this in future versions of this specification.

We plan to add a separate fluff augmentation specification that will introduce random delays.
Optimal delay times depend on the message frequency and patterns.
This delay fluff augmentation specification will be oblivious to the actual message content,
because Waku Dandelion specifications add anonymity on the routing layer.
Still, it is important to note that [Waku2 messages](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md/#payloads) (in their current version) carry an originator timestamp,
which works against fluff phase random delays.
An analysis of the benefits of this timestamp versus anonymity risks is on our roadmap.

By adding a delay, the fluff phase modifies the behaviour of [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md),
which Waku Relay builds upon.

Note: Introducing random delays can have a negative effect on
[peer scoring](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#peer-scoring).

#### Stem Flag

While WAKU2-DANDELION without fluff augmentation does not effect Waku Relay nodes,
messages sent by nodes that only support [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) might be routed through a Dandelion stem without them knowing.
While this improves anonymity, as discussed above, it also introduces additional latency and lightpush nodes cannot opt out of this.

In future versions of this specification we might

- add a flag to [14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md) indicating a message should be routed over a Dandelion stem (opt-in), or
- add a flag to [14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md) indicating a message should _not_ be routed over a Dandelion stem (opt-out), or
- introducing a fork of [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) exclusively used for Dandelion stem.

In the current version, we decided against these options in favour of a simpler protocol and an increased anonymity set.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [Dandelion](https://arxiv.org/abs/1701.04439)
- [Dandelion++](https://arxiv.org/abs/1805.11060)
- [multi-node (botnet) attacker model](../../informational/adversarial-models.md/#multi-node)
- [Waku Relay](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [Waku v2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)
- [d-regular graph](https://en.wikipedia.org/wiki/Regular_graph)
- [Anonymity Trilemma](https://freedom.cs.purdue.edu/projects/trilemma.html)
- [Waku Privacy and Anonymity Analysis](https://vac.dev/wakuv2-relay-anon).
- [On the Anonymity of Peer-To-Peer Network Anonymity Schemes Used by Cryptocurrencies](https://arxiv.org/pdf/2201.11860.pdf)
- [Adversarial Models](../../informational/adversarial-models.md)
- [14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md)
- [29/WAKU-CONFIG](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/29/config.md)
- [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)
