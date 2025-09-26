---
title: WAKU-API
name: Waku API definition
category: Standards Track
status: raw
tags: [reliability, application, api, protocol composition]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
- Oleksandr Kozlov <oleksandr@status.im>
- Prem Chaitanya Prathi <prem@status.im>
- Franck Royer <franck@status.im>
---

## Table of contents

<!-- TOC -->
  * [Table of contents](#table-of-contents)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
  * [Syntax](#syntax)
  * [API design](#api-design)
    * [IDL](#idl)
    * [Primitive types and general guidelines](#primitive-types-and-general-guidelines)
    * [Language mappings](#language-mappings)
    * [Application](#application)
  * [The Waku API](#the-waku-api)
    * [Initialise Waku node](#initialise-waku-node)
      * [Type definitions](#type-definitions)
      * [Function definitions](#function-definitions)
      * [Predefined values](#predefined-values)
      * [Extended definitions](#extended-definitions)
  * [The Validation API](#the-validation-api)
  * [Security/Privacy Considerations](#securityprivacy-considerations)
  * [Copyright](#copyright)
<!-- TOC -->

## Abstract

This document specifies an Application Programming Interface (API) that is RECOMMENDED for developers of the [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md) clients to implement,
and for consumers to use as a single entry point to its functionalities.

This API defines the RECOMMENDED interface for leveraging Waku protocols to send and receive messages. 
Application developers SHOULD use it to access capabilities for peer discovery, message routing, and peer-to-peer reliability.

TODO: This spec must be further extended to include connection health inspection, message sending, subscription and store hash queries.

## Motivation

The accessibility of Waku protocols is capped by the accessibility of their implementations, and hence API.
This RFC enables a concerted effort to draft an API that is simple and accessible, and provides an opinion on sane defaults.

The API defined in this document is an opinionated-by-purpose method to use the more agnostic [WAKU2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md) protocols.

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

- No `default` means that the value is mandatory, meaning a `default` value implies an optional parameter.
- Primitive types are `string`, `int`, `bool`, `enum` and `uint`
- Complex pre-defined types are:
  - `object`: object and other nested types.
  - `array`: iterable object containing values of all the same type.
  - `result`: an enum type that either contains a value or void (success), or an error (failure); The error is left to the implementor.
  - `error`: Left to the implementor on whether `error` types are `string` or `object` in the given language.
- Usage of `result` is RECOMMENDED, usage of exceptions is NOT RECOMMENDED, no matter the language.

TODO: Review whether to specify categories of errors.

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

This API is designed for generic use and ease across all programming languages, for `edge` and `sovereign` type nodes.

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
    type: object
    description: "A Waku node instance."

  NodeConfig:
    type: object
    fields:
      mode:
        type: string
        constraints: [ "edge", "sovereign" ]
        default: "sovereign" # "edge" for mobile and browser devices.
        description: "The mode of operation of the Waku node. Core protocols used by the node are inferred from this mode."
      waku_config:
        type: WakuConfig
        default: TheWakuNetworkPreset
      message_confirmation:
        type: bool
        default: true
        description: "Whether to apply peer-to-peer reliability strategies to confirm that outgoing message have been received by other peers."
      networking_config:
        type: NetworkConfig
        default: DefaultNetworkingConfig 
      eth_rpc_endpoints:
        type: array<string>
        description: "Eth/Web3 RPC endpoint URLs, only required when RLN is used for message validation; fail-over available by passing multiple URLs. Accepting an object for ETH RPC will be added at a later stage."

  WakuConfig:
    type: object
    fields:
      entry_nodes:
        type: array<string>
        default: []
        description: "Nodes to connect to; used for discovery bootstrapping and quick connectivity. enrtree and multiaddr formats are accepted. If not provided, node does not bootstrap to the network (local dev)."
      static_store_nodes:
        type: array<string>
        default: []
        # TODO: confirm behaviour at implementation time.
        description: "The passed nodes are prioritised for store queries."
      cluster_id:
        type: uint
      auto_sharding_config:
        type: AutoShardingConfig
        default: DefaultAutoShardingConfig
        description: "The auto-sharding config, if sharding mode is `auto`"
      message_validation:
        type: MessageValidation
        description: "If the default config for TWN is not used, then we still provide default configuration for message validation." 
        default: DefaultMessageValidation

  NetworkingConfig:
    type: object
    fields:
      listen_ipv4:
        type: string
        default: "0.0.0.0"
        description: "The network IP address on which libp2p and discv5 listen for inbound connections. Not applicable for some environments such as the browser." 
      p2p_tcp_port:
        type: uint
        default: 60000
        description: "The TCP port used for libp2p, relay, etc aka, general p2p message routing. Not applicable for some environments such as the browser."
      discv5_udp_port:
        type: uint
        default: 9000
        description: "The UDP port used for discv5. Not applicable for some environments such as the browser."

  AutoShardingConfig:
    type: object
    fields:
      num_shards_in_cluster:
        type: uint
        description: "The number of shards in the configured cluster; this is a globally agreed value for each cluster."

  MessageValidation:
    type: object
    fields:
      max_message_size:
        type: string
        default: "150 KiB"
        description: "Maximum message size. Accepted units: KiB, KB, and B. e.g. 1024KiB; 1500 B; etc."
      # For now, RLN is the only message validation available
      rln_config:
        type: RlnConfig
        # If the default config for TWN is not used, then we do not apply RLN
        default: none

  RlnConfig:
    type: object
    fields:
      contract_address:
        type: string
        description: "The address of the RLN contract exposes `root` and `getMerkleRoot` ABIs"
      chain_id:
        type: uint
        description: "The chain id on which the RLN contract is deployed"
      epoch_size_sec:
        type: uint
        description: "The epoch size to use for RLN, in seconds"
```

#### Function definitions

```yaml
functions:
  createNode:
    description: "Initialise a Waku node instance"
    parameters:
      - name: nodeConfig
        type: NodeConfig
        description: "The Waku node configuration."
    returns:
        type: result<WakuNode, error>
```

#### Predefined values

```yaml
values:

  DefaultNetworkingConfig:
    type: NetworkConfig
    fields:
      listen_ipv4: "0.0.0.0"
      p2p_tcp_port: 60000
      discv5_udp_port: 9000

  TheWakuNetworkPreset:
    type: WakuConfig
    fields:
      entry_nodes: [ "enrtree://AIRVQ5DDA4FFWLRBCHJWUWOO6X6S4ZTZ5B667LQ6AJU6PEYDLRD5O@sandbox.waku.nodes.status.im" ]
      # On TWN, we encourage the usage of discovered store nodes
      static_store_nodes: []
      cluster_id: 1
      auto_sharding_config:
        fields:
          num_shards_in_cluster: 8
      message_validation: TheWakuNetworkMessageValidation

  TheWakuNetworkMessageValidation:
    type: MessageValidation
    fields:
      max_message_size: "150 KiB"
      rln_config:
        fields:
          contract_address: "0xB9cd878C90E49F797B4431fBF4fb333108CB90e6"
          chain_id: 59141
          epoch_size_sec: 600 # 10 minutes

  # If not preset is used, autosharding on one cluster is applied by default
  # This is a safe default that abstract shards (content topic shard derivation), and it enables scaling at a later stage
  DefaultAutoShardingConfig:
    type: AutoShardingConfig
    fields:
      num_shards_in_cluster: 1

  # If no preset is used, we only apply a max size limit to messages
  DefaultMessageValidation:
    type: MessageValidation
    fields:
      max_message_size: "150 KiB"
      rln_config: none
```

#### Extended definitions

**`mode`**:

If the `mode` set is `edge`, the initialised `WakuNode` MUST mount:

- [LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) as client 
- [FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md) as client
- [STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md) as client
- [METADATA](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/66/metadata.md) as client

And must use mount and use the following protocols to discover peers:

- [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md)

If the `mode` set is `sovereign`, the initialised `WakuNode` MUST mount:

- [RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) as service node
- [FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md) as service node
- [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md) as service node
- [STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md) as client
- [METADATA](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/66/metadata.md) as client and service node

And must use mount and use the following protocols to discover peers:

- [DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md)
- [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md)
- [RENDEZVOUS](https://github.com/waku-org/specs/blob/master/standards/core/rendezvous.md)

`edge` mode SHOULD be used if node functions in resource restricted environment,
whereas `sovereign` SHOULD be used if node has no strong hardware or bandwidth restrictions.

**`message_confirmation`**:

As defined in [P2P-RELIABILITY](/standards/application/p2p-reliability.md).
Proceed with confirmation on whether outgoing messages were received by other nodes in the network.

When set to true, [Store-based reliability for publishing](/standards/application/p2p-reliability.md#1-store-based-reliability-for-publishing) SHOULD be enabled.
In `edge` `mode`, [Retransmit on possible message loss detection](/standards/application/p2p-reliability.md#4-retransmit-on-possible-message-loss-detection) by installing filter subscription(s) matching the content topic(s) used for publishing, MAY be enabled.

## The Validation API

[WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md) is currently the primary message validation mechanism in place.

Work is scheduled to specify a validate API to enable plug-in validation.
As part of this API, it will be expected that a validation object can be passed,
that would contain all validation parameters including RLN.

In the time being, parameters specific to RLN are accepted for the message validation.
RLN can also be disabled.

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
