---
title: MESSAGING-API
name: Messaging API definitions
category: Standards Track
tags: [reliability, application, api, protocol composition]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
- Oleksandr Kozlov <oleksandr@status.im>
- Prem Chaitanya Prathi <prem@status.im>
---

## Abstract

This document specifies an Application Programming Interface (API) that is RECOMMENDED for consumers of the [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md) protocol suite as a single entry point to its functionalities.

The API generalizes the following core protocols provided by Waku:
- [WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md)
- [WAKU2-STORE](../standards/core/store.md)
- [WAKU2-LIGHTPUSH](../standards/core/lightpush.md)
- [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md)

This API extends the core protocols with additional functionality
required for peer-to-peer messaging applications, including:
- Defines a set of events which MAY be used for subscription to track message propagation, node state, and error handling.
- Defines a set of method calls which MAY be used for invocation of protocol operations.
- Abstracts new primitives to be accessed in a RESTful manner via [Waku REST API](https://waku-org.github.io/waku-rest-api/).

This document defines the specifications and guidelines necessary for implementing the API,
ensuring interoperability and consistency across the Waku protocol family.

## Design Requirements
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

### Definitions

This section defines key terms and concepts used throughout this document.
Each term is detailed in dedicated subsections to ensure clarity and consistency in interpretation.

`Multiaddr` - a self-describing format for encoding network address of a remote peer as per [libp2p addressing](https://github.com/libp2p/specs/blob/6d38f88f7b2d16b0e4489298bcd0737a6d704f7e/addressing/README.md) specification.

### Motivation

Real-world application development has exposed challenges in achieving comprehensive usability guarantees for core Waku protocols.
Although recommendations such as [P2P-RELIABILITY](./p2p-reliability.md) have been introduced to enhance protocol robustness,
the fragmented landscape of guidelines and the requirement to re-implement reliability strategies across diverse use cases remain evident.

A significant challenge has been the lack of a unified abstraction for message exchange.
Developers require a straightforward mechanism to send,
receive, and track messages—whether originating from a node or propagated throughout its connected network.

## API design

### Requirements

This API is designed for generic use and ease of implementation across `nwaku`, `js-waku`,
and traditional `REST` architectures.
From this point forward,
relevant code blocks will be provided in `JSON` format, `HTTP` format or by using `TypeScript` syntax.

### Initial configuration

The `Messaging API` is an abstraction layer built upon the basic functionality provided by a node.  
The configuration definitions that follow omit implementation-specific details,
focusing exclusively on settings explicitly required by the `Messaging API`.

```typescript
{
  mode: "edge" | "service";
  clusterId: number;
  shards: number[];
  storeNodes?: string[];
  serviceNodes?: string[];
  bootstrapNodes: string[];
}
```

#### Properties

##### `mode`
This property defines behavior of the node and MUST be specified.

If `edge` selected the node MUST use [WAKU2-LIGHTPUSH](../standards/core/lightpush.md) for sending messages,
and [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) for receiving messages.

If `service` selected the node MUST implement [WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md),
[WAKU2-STORE](../standards/core/store.md) as well as host endpoint for [WAKU2-LIGHTPUSH](../standards/core/lightpush.md) and [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md).

`edge` mode SHOULD be used if node functions in resource restricted environment,
where as `service` SHOULD be used if node has no hard restrictions.

##### `clusterId`
This property MUST be provided.

It signifies which cluster a node MUST be operating at as per [RELAY-SHARDING](https://github.com/waku-org/specs/blob/186ce335667bdfdb6b2ce69ad7b2a3a3791b1ba6/standards/core/relay-sharding.md).

##### `shards`
This property MUST be provided.

An array of shard under a specified cluster that node MUST operate at as per [RELAY-SHARDING](https://github.com/waku-org/specs/blob/186ce335667bdfdb6b2ce69ad7b2a3a3791b1ba6/standards/core/relay-sharding.md).

##### `storeNodes`
A list of `Multiaddr` addresses to remote peers that SHOULD be used for retrieving past messages from [STORE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/13/store.md) protocol.

If not provided, nodes discovered through the network SHOULD be used.

##### `serviceNodes`
A list of `Multiaddr` addresses to remote peers that SHOULD be used for getting resources from the network.

These resources MAY include:
- infrequent [WAKU2-STORE](../standards/core/store.md) queries;
- [WAKU2-LIGHTPUSH](../standards/core/lightpush.md) endpoint for broadcasting data;
- [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) endpoint for receiving data that circulates in the network;

If not provided, nodes discovered through the network SHOULD be used.

##### `bootstrapNodes`
This property MUST be provided.

A list of `Multiaddr` addresses to remote peers that SHOULD be used for any applicable method of discovery that MAY include:
- [EIP-1459](https://eips.ethereum.org/EIPS/eip-1459);
- [WAKU2-DISCV5](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/33/discv5.md);
- [WAKU2-PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/34/peer-exchange.md).

#### Methods / `REST` endpoints

No methods or `REST` endpoints are provided for initial configuration.
Instead, the initial configuration MUST be supplied at node creation time according to the guidelines of the chosen implementation.

### Send
#### Encoder POJO definition
#### Nim / JS definitions
#### REST API endpoint

### Subscribe
#### Decoder POJO definition
#### Nim / JS
#### REST API endpoint

### Message storage
#### Nim / JS
#### REST API endpoint

### Health indicator

The `Health Indicator API` SHOULD be used to monitor the health of a node operating under the `Messaging API`.
The specific criteria for determining the health state MAY vary depending on the protocols the node utilizes.

We define three health states:
- unhealthy;
- minimally healthy;
- sufficiently healthy;

#### Health states

##### `unhealthy`
No connections to service nodes are available, regardless of protocol.
No peers are present in the Relay mesh.

##### `minimally healthy`
The node has one connection to a service node implementing Filter AND one connection to a service node implementing LightPush.
Relay mesh has connection to at least four other peers.

##### `sufficiently healthy`
The node has at least two connections to service nodes implementing Filter AND at least two connections to service nodes implementing LightPush.
Relay mesh has connection to at least six other peers.

#### Programmatic API

```typescript
enum HealthStatus {
  Unhealthy = "Unhealthy",
  MinimallyHealthy = "MinimallyHealthy",
  SufficientlyHealthy = "SufficientlyHealthy"
}

node.healthIndicator in HealthStatus;
node.healthIndicator.toString(); // -> HealthStatus string value
```

#### REST API

```http
GET /health HTTP/1.1
Accept: application/json
```

Example of a response:
```json
{
  status: "MinimallyHealthy"
}
```

### Event source

One component of the `Messaging API` is an event emitter that SHOULD be used to track various internal activities.
Consumers can subscribe to this event source to monitor message propagation, network changes, or the health status of the node.

For accessing events via the `REST API`, long polling SHOULD be used.

#### Events

When an event is fired, it indicates that a change has already occurred.
Therefore, querying the original value storage will yield the same value as the one that was fired.

##### `health:change`
This event MUST be emitted whenever a change is detected by the `Health Indicator API`.
The payload of the event is the new health status value.

Programmatic API:
```typescript
node.events.addEventListener("health:change", (event: CustomEvent<HealthStatus>) => {
  // event.detail holds the HealthStatus value
  console.log(event.detail);
});
```

For the `REST API`, long polling over the `Health Indicator API` SHOULD be used:
```http
GET /health HTTP/1.1
Accept: application/json
```

##### `network:change`
This event MUST be emitted when the node transitions between offline and online states.
- online: The node has stable Internet access AND at least one connection to a remote peer.
- offline: The node lacks Internet connectivity OR has no connections to remote peers.

Programmatic API:
```typescript
enum NetworkState {
  Online = "Online",
  Offline = "Offline"
}

node.events.addEventListener("network:change", (event: CustomEvent<NetworkState>) => {
  // event.detail holds NetworkState
  console.log(event.detail);
});
```

`REST API`:
```http
GET /network HTTP/1.1
Accept: application/json
```

Example of a response:
```json
{
  status: "Online"
}
```

##### `message:change`
This event indicates changes that occur to a message that is being tracked by

Programmatic API:
```typescript

```

For the `REST API`, long polling over the `Message Store API` SHOULD be used:
```http
GET /messages-store HTTP/1.1
Accept: application/json
```

Example of a response:
```json
{
  messages: [
    {
      hash: "bab8df...";
      requestId: "e180f378-...-b29c90d2ed7c";
      states: ["FILTER", "STORE"];
    }
  ]
}
```

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
