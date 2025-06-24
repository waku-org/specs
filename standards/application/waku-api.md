---
title: MESSAGING-API
name: Messaging API definition
category: Standards Track
tags: [reliability, application, api, protocol composition]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
- Oleksandr Kozlov <oleksandr@status.im>
- Prem Chaitanya Prathi <prem@status.im>
- Franck Royer <franck@status.im>
---

## Table of contents

TODO

## Abstract

This document specifies an Application Programming Interface (API) that is RECOMMENDED for developers of the [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md) clients to implement,
and for consumers to use as a single entry point to its functionalities.

This API defines the RECOMMENDED interface for leveraging Waku protocols to send and receive messages. 
Application developers SHOULD use it to access capabilities for peer discovery, message routing, and peer-to-peer reliability.

## Motivation

The accessibility of Waku protocols is capped by the accessibility of their implementations, and hence API.
This RFC enables a concerted effort to draft an API that is simple and accessible, and provides an opinion on sane defaults.

This effort is best done in an RFC, allowing all current implementors to review and agree on it. 

The API defined in this document is an opinionated-by-purpose method to use the more agnostic [WAKU2]() protocols.

## Syntax

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

### Definitions

This section defines key terms and concepts used throughout this document.
Each term is detailed in dedicated subsections to ensure clarity and consistency in interpretation.

`Multiaddr` - a self-describing format for encoding network address of a remote peer as per [libp2p addressing](https://github.com/libp2p/specs/blob/6d38f88f7b2d16b0e4489298bcd0737a6d704f7e/addressing/README.md) specification.

## API design

### IDL

A custom Interface Definition Language (IDL) in YAML is used to define the Waku API.
Existing IDL Such as OpenAPI, AsyncAPI or WIT do not exactly fit the requirements for this API.
Hence, instead of having the reader learn a new IDL, we propose to use a simple IDL with self-describing syntax.

An alternative would be to choose a programming language. However, such choice may express unintended opinions on the API.

#### Guidelines

- No `default` means that the value is mandatory
- Primitive types are `string`, `int`, `bool`, `enum` and `uint`
- Complex pre-defined types are:
  - `struct`: object and other nested types
  - `option`: a value that can be set or left null
  - `array`: iterable object containing values of all the same type
  - `Multiaddr`: a libp2p multiaddress; may be an object or a string, most idiomatic approach depending on the language

### Application

This API is designed for generic use and ease across all programming languages and traditional `REST` architectures.

## The Waku API

```yaml
api_version: "0.0.1"
library_name: "waku"
description: "Waku: a private and censorship-resistant message routing library."
```
### Language Mappings

How the API definition should be translated to specific languages.

```yaml
language_mappings:
  rust:
    naming_convention:
      - functions: "snake_case"
      - variables: "snake_case"
      - types: "PascalCase"
    error_handling: "Result<T, E>"
    async_pattern: "tokio"
    
  golang: 
    naming_convention:
       - functions: "snake_case"
       - variables: "snake_case"
       - types: "PascalCase"
    error_handling: "error_return"
    package_name: "waku"
    
  c:
    naming_convention: "snake_case"
    prefix: "waku_"
    error_handling: "error_codes"
    
  typescript:
    naming_convention:
      - functions: "camelCase"
      - variables: "camelCase"
      - types: "PascalCase"
    error_handling: "Promise<T>"
    module_type: "esm"
```


### Type definitions

```yaml
types:
  Config:
    type: struct
    fields:
      mode: 
        type: string
        constraints: ["edge", "relay"]
        description: "The mode of operation of the Waku node. Core protocols used by the node are inferred from this mode."
      network_config:
        type: NetworkConfig
        default: TheWakuNetworkPreset
      active_relay_shards:
        type: array<uint>
        constraints: mode == "relay"
        default: []
        description: "The shards for relay to subscribe to and participate in."
      store_confirmation:
        type: bool
        default: false
        description: "No-payload store hash queries are made to confirm whether outbound messages where received by remote store node."

  NetworkConfig:
    type: struct
    fields:
      boostrap_nodes:
        type: array<string>
        default: ""
        description: "Bootstrap nodes, entree and multiaddr formats are accepted."
      static_store_nodes:
        type: array<string>
        default: []
        description: "Only the passed nodes are used for store queries, discovered store nodes are discarded."
      cluster_id:
        type: uint
        default: 1
      sharding_mode:
        constraints: ["auto", "static"]
      auto_sharding_config:
        type: option<AutoShardingConfig>
        default: none
        description: "The auto-sharding config, if sharding mode is `auto`"

  AutoShardingConfig:
    type: struct
    fields:
      numShardsInCluster:
        type: uint
        description: "The number of shards in the configured cluster; this is a globally agreed value for each cluster."
```

### Initialise Waku Node

TODO: define WakuNode?

```yaml
functions:
  init:
    description: "Initialise the waku node"
    parameters:
        - name: config
          type: Config
          description: "The Waku node configuration."
    returns:
        type: result<void, string>
```

#### Functionality / Additional Information / Specific Behaviours

If the node is operating in `edge` mode, it MUST:

- Use [LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) to send messages
- Use [FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md) to receive messages
- Use [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md#abstract) to discover peers
- Use [STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md) as per [WAKU-P2P-RELIABILITY](/standards/application/p2p-reliability.md)

If the node is configured in `relay` mode, it MUST:

- Use [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) protocol.
- Host endpoints for [LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) and [FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md).
- Serve the [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md) protocol.

`edge` mode SHOULD be used if node functions in resource restricted environment,
whereas `relay` SHOULD be used if node has no hard restrictions.

#### Default Values

```yaml
values:
  TheWakuNetworkPreset:
    type: NetworkConfig 
    fields:
      bootstrap_nodes: ["enrtree://AIRVQ5DDA4FFWLRBCHJWUWOO6X6S4ZTZ5B667LQ6AJU6PEYDLRD5O@sandbox.waku.nodes.status.im"]
      static_store_nodes: #TODO: enter sandbox store nodes multiaddr
      cluster_id: 1
      sharding_mode: "auto"
      auto_sharding_config: TheWakuNetworkAutoShardingConfig
  TheWakuNetworkAutoShardingConfig:
    type: AutoShardingConfig
    fields:
      numShardsInCluster: 8
```

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
