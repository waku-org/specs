---
title: WAKU2-INCENTIVIZATION
name: Incentivization for Waku Light Protocols
category: Standards Track
tags: [incentivization]
editor: Sergei Tikhomirov <sergei@status.im>
contributors:
---

## Abstract

This document describes an approach to incentivization of Waku request-response protocols.
Incentivization is necessary for economically sustainable growth of Waku.
In an incentivized request-response protocol, only eligible (e.g., paying) clients receive the service.
Clients include eligibility proofs in their requests.

Eligibility proofs are designed to be used in multiple Waku protocols, such as Store, Lightpush, and Filter.
Lightpush is planned to become the first Waku protocol with an incentivization component.
In particular, a Lightpush client will be able to publish messages without their own RLN membership.
Instead, the client would pay the server for publishing the client's message using the server's RLN proof.
We will discuss a proof-of-concept implementation of this incentivization component in a later section.

## Background / Rationale / Motivation

Decentralized protocols require incentivization to be economically sustainable.
While some aspects of a P2P network can successfully operate in a tit-for-tat model,
we believe that nodes that run the protocol in good faith need to be tangibly rewarded.
Motivating servers to expand resources on handling clients' requests allows us to scale the network beyond its initial altruism-based phase.

Incentivization is not necessarily limited to monetary rewards.
Reputation may also play a role.
For Waku request-response (i.e., client-server) protocols, we envision a combination of monetary and reputation-based incentivization.
See a [write-up on incentivization](https://github.com/waku-org/research/blob/1e3ed6a5cc47e6d1e7cb99271ddef9bf38429518/docs/incentivization.md) for our high-level reasoning on the topic.

## Theory / Semantics

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Consider a request-response protocol with two roles: a client and a server.
A server MAY indicate to a client that it expects certain eligibility criteria to be met.
In that case, a client MUST provide a valid eligibility proof as part of its request.

Forms of eligibility proofs include:

- Proof of payment: for paid non-authenticated requests. A proof of payment, in turn, may also take different forms, such as a transaction hash or a ZK-proof. In order to interpret a proof of payment, the server needs information about its type.
- Proof of membership: for services for a predefined group of users. An example use case: an application developer pays in bulk for their users' requests. A client then prove that they belong to the user set of that application. Rate limiting in Waku RLN Relay is based on a similar concept.
- Service credential: a proof of membership in a set of clients who have prepaid for the service (which may be considered a special case of proof of membership).

Upon a receiving a request:

- the server SHOULD check if the eligibility proof is included and valid;
- if that proof is absent or invalid, the server SHOULD send back a response with a corresponding error code and an error description;
- if the proof is valid, the server SHOULD send back the response that the client has requested.

Note that the protocol does not ensure atomicity.
It is technically possible for a server to fail to respond to an eligible request (in violation of the protocol).
Addressing this issue is left for future work.

## Wire Format Specification / Syntax

A client includes an `EligibilityProof` in its request.
A server includes an `EligibilityStatus` in its response.

```protobuf
syntax = "proto3";

message EligibilityProof {
  optional bytes proof_of_payment = 1;  // e.g., a txid
  // may be extended with other eligibility proof types, such as:
  //optional bytes proof_of_membership = 2;  // e.g., an RLN proof
}

message EligibilityStatus {
  optional uint32 status_code = 1;
  optional string status_desc = 2;
}
```

We include the `other_eligibility_proof` field in `EligibilityProof` to reflect other types of eligibility proofs that could be added to the protocol later.

## Implementation in Lightpush (PoC version)

This Section describes a proof-of-concept (PoC) implementation of incentivization in the Lightpush protocol.
Note: this section may later be moved to Lightpush RFC.

Lightpush is one of Waku's request-response protocols.
A Lightpush client sends a request to the server containing the message to be published to the Waku network on the client's behalf.
A Lightpush server responds to indicate whether the client's message was successfully published.
See [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) for the definitions of `PushRequest` and `PushResponse`.

The PoC Lightpush incentivization makes the following simplifying assumptions:

- the client knows the server's on-chain address `A` (likely on an L2 network);
- the client and the server have agreed on a constant price `p` per published message.

To publish a message, the client:

1. pays `p` to the server's address `A` with an on-chain transaction;
2. waits until the transaction is confirmed with identifier `txid`;
3. includes `txid` in the request as a proof of payment.

It is the server's responsibility to keep track of the `txid`s from prior requests and to make sure they are not reused.

Note that `txid` may not always be practical as proof of payment due to on-chain confirmation latency.
To address this issue, future versions of the protocol may involve bulk payments, that is, paying for multiple requests in one transaction.
In that scheme, after a bulk payment is made, the client will not have to face on-chain latency for each new prepaid request.

### Wire Format Specifications for Lightpush PoC incentivization

#### Request

We extend `PushRequest` to include an eligibility proof:

```protobuf
message PushRequest {
  string pubsub_topic = 1;
  WakuMessage message = 2;
  // numbering gap left for non-eligibility-related protocol extensions
  + optional bytes eligibility_proof = 10;
}
```

An example of usage with `txid` as a proof of payment:

```protobuf
PushRequest push_request {
  pubsub_topic: "example_pubsub_topic"
  message: "example_message"
  eligibility_proof: {
    proof_of_payment: 0xabc123  // txid for the client's payment
    // eligibility proofs of other types are not included
  };
}
```

#### Response

We extend the `PushResponse` to indicate the eligibility status:

```protobuf
message PushResponse {  
    bool is_success = 1;
    // Error messages, etc
    string info = 2;
  + EligibilityStatus eligibility_status = 3;
}
```

Example of a response if the client is eligible:

```protobuf
PushResponse response_example {
  is_success: true
  info: "Request successful"
  eligibility_status: {
    status_code: 200
    status_desc: "OK"
    }
  }
```

Example of a response if the client is not eligible:

```protobuf
PushResponse response_example {
  is_success: false
  info: "Request failed"
  eligibility_status: {
    status_code: 402
    status_desc: "PAYMENT_REQUIRED"
  }
}
```

## Security/Privacy Considerations

Eligibility proofs may reveal private information about the client.
In particular, a transaction identifier used as a proof of payment links the client's query to their on-chain activity.
Potential countermeasures may include using one-time addresses or ZK-based privacy-preserving protocols.

## Limitations and Future Work

This document is intentionally simplified in its initial version.
It assumes a shared understanding of prices and the blockchain addresses of servers.
Additionally, the feasibility of paying for each query is hindered by on-chain fees and confirmation delays.

We will address these challenges as the specification evolves alongside the corresponding PoC implementation.
The following ideas will be explored:

- Batch Payment: instead of paying for an individual query, the client would make a consolidated payment for multiple messages.
- Price Negotiation: rather than receiving prices off-band, the client would engage in negotiation with the server to determine costs.
- Dynamic Pricing: the price per message would be variable, based on the total size (in bytes) of all received messages.
- Subscriptions: the client would pay for a defined time period during which they can query any number of messages, subject to DoS protection.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

### normative

- A high-level [incentivization outline](https://github.com/waku-org/research/blob/master/incentivization.md)
- [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) (for Lightpush-specific sections)

### informative

RFCs of request-response protocols:

- [12/WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md)
- [13/WAKU2-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md)
- [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)

RFCs of Relay and RLN-Relay:

- [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [17/WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)
