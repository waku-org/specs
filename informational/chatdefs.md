---
title: CHAT-DEFINITIONS
name: Shared definitions for Chat Protocols
category: Informational
tags: definitions, terminology, reference
editor: Jazzz
contributors:
---

## Abstract

This specification establishes a common vocabulary and standardized definitions for terms used across chat protocol specifications. It serves as a normative reference to ensure consistent interpretation of terminology across application level specifications.

## Background / Rationale / Motivation

Chat protocol specifications often employ similar terminology with subtle but important semantic differences. The absence of standardized definitions leads to ambiguity in specification interpretation and implementation inconsistencies across different protocol variants.

Establishing a shared definitions document provides several benefits:

1. **Semantic Consistency**: Ensures uniform interpretation of common terms across multiple specifications
2. **Implementation Clarity**: Reduces ambiguity for protocol implementers
3. **Specification Quality**: Enables more precise and concise specification language through reference to established definitions
4. **Ecosystem Coherence**: Promotes consistent understanding within the chat protocol ecosystem.

This specification provides normative definitions that other chat protocol specifications MAY reference to establish precise semantic meaning for common terminology.

## Theory / Semantics

### Definition Categories

Terms are organized into the following categories for clarity and ease of reference:

- **Roles**: Defined entity types that determine how a participant behaves within a communication protocol.
- **Message Types**: Classifications and categories of protocol messages
- **Transports**: Abstract methods of transmitting payloads
## Definitions

### Accounts + Identity

### Roles

**Sender**: [TODO]

**Recipient**: [TODO]

**Participant**: [TODO]

**Operator**: [TODO]

### Message Types

**Payload**: [TODO]

**Frame**: [TODO]

**Content**: [TODO]

**ContentType**: [TODO]

**Delivery Acknowledgement**: [TODO]

**Invite**: [TODO]


### Transports

**Out of Band**: [TODO]

**Delivery Service**: [TODO]


### TBD

**Client**: [TODO]

**Application**: [TODO]




## Wire Format Specification / Syntax

This specification does not define wire format elements. All definitions are semantic and apply to the interpretation of terms used in other specifications that do define wire formats.

## Security/Privacy Considerations

This specification defines terminology only and does not introduce security or privacy considerations beyond those present in specifications that reference these definitions.

The definitions provided in this specification do not alter the security or privacy properties of implementations that adopt the defined terminology.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

[RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.