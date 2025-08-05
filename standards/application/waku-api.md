---
title: WAKU-API
name: Waku API definition
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

The API defined in this document is an opinionated-by-purpose method to use the more agnostic [WAKU2]() protocols.

## Syntax

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## API design

### IDL

A custom Interface Definition Language (IDL) in YAML is used to define the Waku API.
Existing IDL Such as OpenAPI, AsyncAPI or WIT do not exactly fit the requirements for this API.
Hence, instead of having the reader learn a new IDL, we propose to use a simple IDL with self-describing syntax.

An alternative would be to choose a programming language. However, such choice may express unintended opinions on the API.

### Primitive types and general guidelines

- No `default` means that the value is mandatory
- Primitive types are `string`, `int`, `bool`, `enum` and `uint`
- Complex pre-defined types are:
  - `struct`: object and other nested types.
  - `option`: a value that can be set or left null.
  - `array`: iterable object containing values of all the same type.
  - `multiaddr`: a libp2p multiaddress; may be an object or a string, most idiomatic approach depending on the language.
  - `result`: an enum type that either contain a return value (success), or an error (failure); The error is left to the implementor.
  - `error`: Left to the implementor on whether `error` types are `string` or `object` in the given language.
- Usage of `result` is RECOMMENDED, usage of exceptions is NOT RECOMMENDED, no matter the language.

### Language mappings

How the API definition should be translated to specific languages.

```yaml
language_mappings:
  typescript:
    naming_convention:
      - functions: "camelCase"
      - variables: "camelCase"
      - types: "PascalCase"
  nim:
    naming_convention:
      - functions: "camelCase"
      - variables: "camelCase"
      - types: "PascalCase"
```

### Application

This API is designed for generic use and ease across all programming languages, for `edge` and `relay` type nodes.

## The Waku API

```yaml
api_version: "0.0.1"
library_name: "waku"
description: "Waku: a private and censorship-resistant message routing library."
```

### Initialise Waku node

#### Type definitions

```yaml
types:
  WakuNode:
    type: struct
    description: "A Waku node instance."
  
  Config:
    type: struct
    fields:
      mode: 
        type: string
        # For now, a mode **must** be passed by the developer
        constraints: ["edge", "relay"]
        description: "The mode of operation of the Waku node. Core protocols used by the node are inferred from this mode."
      network_config:
        type: NetworkConfig
        default: TheWakuNetworkPreset
      store_confirmation:
        type: bool
        # Until further dogfooding, assuming default false, usage of SDS should be preferred
        default: false
        description: "No-payload store hash queries are made to confirm whether outbound messages where received by remote store node."

  NetworkConfig:
    type: struct
    fields:
      boostrap_nodes:
        type: array<string>
        # Default means the node does not bootstrap, it is not ideal but practical for local development
        # TODO: get feedback
        default: ""
        description: "Bootstrap nodes, entree and multiaddr formats are accepted."
      static_store_nodes:
        type: array<string>
        default: []
        description: "Only the passed nodes are used for store queries, discovered store nodes are discarded."
      cluster_id:
        type: uint
      sharding_mode:
        constraints: ["auto", "static"]
        # The default network config is TheWakuNetwork, but if a dev override it, then we still provide a sharding default
        default: "auto"
      auto_sharding_config:
        type: option<AutoShardingConfig>
        default: DefaultAutoShardingConfig
        description: "The auto-sharding config, if sharding mode is `auto`" 

  AutoShardingConfig:
    type: struct
    fields:
      numShardsInCluster:
        type: uint
        description: "The number of shards in the configured cluster; this is a globally agreed value for each cluster."
```

#### Function definitions

```yaml
functions:
  create:
    description: "Initialise a Waku node instance"
    parameters:
      - name: config
        type: Config
        description: "The Waku node configuration."
    returns:
        type: result<WakuNode, error>
```

#### Predefined values

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
  
  # If TheWakuNetworkPreset is not used, autosharding is one cluster is applied by default
  # This is a safe default that abstract shards (content topic shard derivation), and enables scaling at a later stage
  DefaultAutoShardingConfig:
    type: AutoShardingConfig
    fields:
      numShardsInCluster: 1
```

#### Extended definitions

If the `mode` set is `edge`, the initialised `WakuNode` MUST mount:

- [LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) as client 
- [FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md) as client
- [STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md) as client

And must use mount and use the following protocols to discover peers:

- [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md)

If the `mode` set is `relay`, the initialised `WakuNode` MUST mount:

- [RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) as service node
- [FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md) as service node
- [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md) as service node
- [STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md) as client

And must use mount and use the following protocols to discover peers:

- [DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md)
- [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md)
- [RENDEZVOUS](https://github.com/waku-org/specs/blob/master/standards/core/rendezvous.md)

`edge` mode SHOULD be used if node functions in resource restricted environment,
whereas `relay` SHOULD be used if node has no strong hardware or bandwidth restrictions.

## The Validation API

[RLN Relay]() is currently the primary message validation mechanism in place.

Work is scheduled to specify a validate API to enable plug-in validation.
As part of this API, it will be expected that an validation object can be passed,
that would contain all validation parameters including RLN.

In the time being, we 

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
