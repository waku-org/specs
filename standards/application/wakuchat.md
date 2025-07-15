---
title: WakuChat
name: A private decentralized messaging protocol for multiple usecases.
category: Standards Track
status: raw
tags: chat
editor: Jazz Alyxzander<jazz@status.im>
contributors:
---
# Abstract

This specification defines a modular communication protocol designed to provide a privacy focused approach to common messaging patterns. The protocol employs a lean root layer that defers to independent sub-protocols for defining communication behavior, with versioning as a first-class concept for handling protocol evolution.

# Background / Rationale / Motivation

Traditionally communication protocols face several critical challenges as they grow. 

Different communication scenarios have vastly different requirements. A protocol optimized for high-volume public broadcasts performs poorly for intimate encrypted conversations, and vice versa. Monolithic protocols cannot optimize for these diverse use cases simultaneously. 

Once widely deployed, communication protocols become difficult to modify. Even minor changes can break compatibility across clients. As a result versioning becomes a complex negotiation problem, which makes deploying protocol updates difficult. At each stage protocols become increasingly difficult to modify, which slows down forward progress and eventually leads to ossification. 

What is desired is integral protocol resiliency by embedding a versioning strategy from the beginning.

# Theory / Semantics

This protocol is a lean orchestration framework which provides the backbone for smaller independent "Conversation" sub-protocols. Conversation protocols completely define a communication channel. This root protocol provides common functionality to support a wide array of communication use cases, and the remaining functionality is deferred to Conversation protocols to define.

This specification outlines how clients can initiate messages with one another, determine how to decode/decrypt incoming messages and support a wide range of communication patterns. 

 ```mermaid
sequenceDiagram
    actor S as Saro 
    participant SI as Saro DefaultInbox 
    participant C as Convo 
    participant RI as Raya DefaultInbox 
    actor R as Raya 


    Note over SI,RI: All clients subscribe to their default Inbox

    SI ->> S: Subscribe
    RI ->> R: Subscribe

    Note over R: Key Information is exchanged OOB 
    
    Note over S: Conversation is created
    C ->> S : Subscribe
    S ->> RI : Send Invite `I1`
    S ->> C : Send Message `M1`

    RI --) R : Recv `I1`
    Note over R: Conversation is joined
    C ->> R : Subscribe
    C --) R: Recv `M1`

    R ->> C: Send M2
    C --) S: Recv M2
 ```


## Conversations 
ConversationTypes are standalone methods to send and receive messages. While a full service "chat" protocol needs to provide additional features such as identity management, message history, backups and versioning, conversations are more narrowly scoped.

They can be created permissionlessly within the WakuChat Protocol, as there is no required registration. Developers are free to use any given conversation type as long as all intended participants support it.



ConversationTypes MUST implement the [Conversations](./conversations.md) specification in order to maintain compatibility. 


## Versioning Strategy

Versioning is one of the hardest problems in decentralized messaging. Keeping a network of decentralized applications and clients up-to-date and version compatible requires careful planning and orchestration. WakuChat's approach to versioning is avoid these problems by removing the issue.

In WakuChat incompatible versions of a conversation protocol result in distinct conversationTypes (E.g. PrivateV1 vs PrivateV3). That is; breaking changes are not distinguished by client version, but by ConversationType. This opinionated approach has some interesting outcomes:

- Compatibility is not determined by the version of a client, but rather by which types the participants supported.
This removes the requirement for all clients to be running the same version, which allows faster roll out of features. Fine grained negotiation of supported types embraces the decentralized nature of the clients and avoids headaches of keeping clients in lockstep.

- Upgrading to a new ConversationType is equivalent to negotiating a new Conversation and cleaning up the old one.
This reuses the existing conversation initiation mechanism, which reduces complexity.

- Forces a decoupling between User-level-conversations and the conversationTypes which provide transport. This makes it easier to roll out changes.

Individual conversationTypes can implement functionality to migrate participants to a conversation, but that is deferred to contributors. Individual conversationType are free to choose an upgrade plan that makes sense for their usecases.


## Default Inbox
There exists a circular dependency in initializing a Conversation. Conversations require an established conversation to send an invitation, yet at the beginning no channel exists between contacts. 

To resolve this all clients MUST implement a default [Inbox](./inbox.md) to receive initial messages. The default inbox allows clients to discover new conversations asynchronously without prior coordination. By listening in a static location know to senders, clients can always receive new invitations.

The default inbox MUST be configured with the parameters:
- **inbox_addr:** `client_address`

One important property of the Inbox is that it can receive different payload types in a single location, by decoupling payload type from the associated contact_topic. New Conversation types will invariably create new payload types. Multiplexing payload types into a single inbox means the default inbox can be reused for undefined future conversationTypes.  

The Default Inbox having public visibility does not preclude Clients having other Inboxes with different levels of visibility. 

As the clients address is directly linked to Default Inbox, this pathway SHOULD only be used as a last resort.  


## Envelopes
As clients can support multiple sub-protocols simultaneously, a mechanism is required to determine how decode payloads.

To process a  payload, a client must know how a payload was encoded/encrypted and what encryption state to use to decrypt it.

**Encoding**
One approach to tracking payload types is to encode it in the content topic. 
Clients can then infer how to decode the payload, based on where it was received. 
The downside is that only one type of payload can be received in a content topic. 
Deploying new types requires a new content topic, which may go unnoticed to clients if they are running an older version or do not support the protocol. 
Even if the client cannot read the payload, knowing it exists can signal to developers a need to update their applications to support the new types; reducing interaction failure and catching errors.

**Encryption State**
To determine which encryption state to use in decryption, applications must attach identifying information. One method to achieve this and ensure privacy, is  shield the required identifying information from public view using asymmetric encryption. However double encryption is expensive, and more opportunity for security faults.  


### Conversation Hinting

Conversation hinting is a mechanism to provide recipients with the necessary information, without exposing conversational metadata.  

Naively, conversation_ids are sufficient to allow clients to determine how to decrypt a payload. As the conversation_id corresponds to the ConversationType which defines how payloads are encdoded, and the encryption state is linked to the specific identifier. However this would leak Conversation metadata, as observers could determine which payloads belonged together into a group. Instead a hint is used which allows clients to check if this payload is of interest. A ConversationHint does not contain identifying information. 

ConversationHints are computed by using a salted hash of the `conversationId`. specifically defined as `lowercase_hex(blake2s(salt || conversation_id))`.

Clients can check if a conversation_hint is of interest to them, then retrieve the associated encryption state. Payloads with an unknown hint can be safely disregarded.


## Wire Format Specification / Syntax

The wire format is specified using protocol buffers v3.

```protobuf

message WapEnvelopeV1 {
    
    // Indicates which conversation this message belongs to without revealing metadata 
    string conversation_hint = 1;
    // Used to compute the conversation_hint
    uint32 salt = 2;           
    // Data encoded according to the corresponding ConversationType
    bytes payload = 3;
}

```

## Implementation Suggestions (optional)

### Application Interop

Developers wishing to interop with other projects will need to ensure they have overlapping support of ConversationTypes.

### User level Conversations

Application developers should maintain standalone identifiers for user-level conversations that are separate from the protocol-level conversation_id. A single logical conversation from the user's perspective may utilize multiple ConversationTypes over time. As versions are considered different types, the underlying relationship is many to one. By maintaining application-level conversation identifiers, developers can provide users with consistent conversation continuity while the underlying protocol mechanisms handle version transitions and security upgrades transparently.


## Security/Privacy Considerations

Payloads inherit the privacy and security properties of the ConversationType used to send them. Please refer to the corresponding specifications when analyzing properties. 

### Default Inbox Privacy
Payloads sent to the default inbox are linkable to an client (as it is derived from the clients address). This means that if a target client address is known to an observer, they can determine if any payloads were sent to the target using the default inbox.  In this case the Envelopes contain no sender information, so this does not leak social graph information.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

A list of references.
