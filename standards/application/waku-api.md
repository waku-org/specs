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
This RFC enables a concerted effort to draft an API that is simple and accessible, and opiniate on sane defaults.

This effort is best done in an RFC, allowing all current implementors to review and agree on it. 

The API defined in this document is an opiniated-by-purpose method to use the more agnostic [WAKU2]() protocols.

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
- Primitive types are `string`, `int`, `bool`, `uint`, and `pointer`
- Complex pre-define types are `struct`, `option` and `array`
- Primitive types are preferred to describe the API for simplicity, the implementator may prefer a native type (e.g. `string` vs `Multiaddr` object/struct)

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
        constraints: ["client", "server"]
        description: "The mode of operation of the Waku node.
                      Waku core protocols used by the node are inferred from this mode."
      network_config:
        type: struct
        fields:
          boostrap_servers:
            type: array<string>
            default: "enrtree://AIRVQ5DDA4FFWLRBCHJWUWOO6X6S4ZTZ5B667LQ6AJU6PEYDLRD5O@sandbox.waku.nodes.status.im"
            description: "Bootstrap nodes. entree, ENRs list formats are accepted.
                          This allows the node to discover server nodes."
            examples:
              ex1: "enrtree://AIRVQ5DDA4FFWLRBCHJWUWOO6X6S4ZTZ5B667LQ6AJU6PEYDLRD5O@sandbox.waku.nodes.status.im"
              ex2: ["enr:-QESuED0qW1BCmF-oH_ARGPr97Nv767bl_43uoy70vrbah3EaCAdK3Q0iRQ6wkSTTpdrg_dU_NC2ydO8leSlRpBX4pxiAYJpZIJ2NIJpcIRA4VDAim11bHRpYWRkcnO4XAArNiZub2RlLTAxLmRvLWFtczMud2FrdS5zYW5kYm94LnN0YXR1cy5pbQZ2XwAtNiZub2RlLTAxLmRvLWFtczMud2FrdS5zYW5kYm94LnN0YXR1cy5pbQYfQN4DgnJzkwABCAAAAAEAAgADAAQABQAGAAeJc2VjcDI1NmsxoQOTd-h5owwj-cx7xrmbvQKU8CV3Fomfdvcv1MBc-67T5oN0Y3CCdl-DdWRwgiMohXdha3UyDw","enr:-QEkuED9X80QF_jcN9gA2ZRhhmwVEeJnsg_Hyg7IFCTYnZD0BDI7a8HArE61NhJZFwygpHCWkgwSt2vqiABXkBxzIqZBAYJpZIJ2NIJpcIQiQlleim11bHRpYWRkcnO4bgA0Ni9ub2RlLTAxLmdjLXVzLWNlbnRyYWwxLWEud2FrdS5zYW5kYm94LnN0YXR1cy5pbQZ2XwA2Ni9ub2RlLTAxLmdjLXVzLWNlbnRyYWwxLWEud2FrdS5zYW5kYm94LnN0YXR1cy5pbQYfQN4DgnJzkwABCAAAAAEAAgADAAQABQAGAAeJc2VjcDI1NmsxoQPFAS8zz2cg1QQhxMaK8CzkGQ5wdHvPJcrgLzJGOiHpwYN0Y3CCdl-DdWRwgiMohXdha3UyDw"]
          static_servers:
            type: array<string>
            default: []
            description: "Array of server nodes' multiaddresses. Discovered nodes may also be considered."
            examples:
              ex1: ["/dns4/node-01.do-ams3.waku.sandbox.status.im/tcp/30303/p2p/16Uiu2HAmNaeL4p3WEYzC9mgXBmBWSgWjPHRvatZTXnp8Jgv3iKsb"]
              ex2: ["/dns4/node-01.do-ams3.waku.sandbox.status.im/tcp/8000/wss/p2p/16Uiu2HAmNaeL4p3WEYzC9mgXBmBWSgWjPHRvatZTXnp8Jgv3iKsb"]
          clusterId:
            type: uint
            constraints: mode == "server"
            default: 1
            description: "The cluster subscribed to and participating in. A cluster is composed by multiple shards."
          shards:
            type: array<uint>
            constraints: mode == "server"
            default: [0, 1, 2, 3, 4, 5, 6, 7]
            description: "The shards subscribed to and participating in."
  ErrorCode:
    type: uint
    constraints: [0 (Ok), 1 (Error), 2 (MissingCallback), 3 (WakuNotResponding)]
    description:
      - 0: The function call was successful and finished on time.
      - 1: Generic error when invoking a particular function. The error detail will be given in the msg callback field.
      - 2: If, when invoking a certain function, a callback is not set properly.
      - 3: When the waku node is blocked in a certain callback, or due to other reason, for too long.

  WakuCallback:
    type: pointer
    description: Generic callback used across the Waku API.
                 Notice that this callback is invoked within the Waku thread and should not be blocked for more than 20 seconds.
                 In case of the Waku thread is blocked for too long,
                 a WakuNotResponding error will be dispatched through the
                 waku_state_event_handler (see the init function definition.)
    fields:
      callerRet:
        type: ErrorCode
      msg:
        type: string
        default: ""
        description: "Information provided to the Waku API integrator"

```

### Initialise Waku

```yaml
functions:
  init:
    description: "Initialise the waku instance"
    parameters:
        - name: config
          type: Config
          description: "The Waku configuration."
        - name: callback
          type: WakuCallback
          description: Gives feedback in case of failure when invoking the function.
        - name: waku_state_event_handler
          type: WakuCallback
          description: Callback that is invoked when there is a Waku state change.
                       For example:\
                       - waku goes online/offline.
                       - waku is not responding for too long.
    returns:
        type: pointer
        description: On success, returns a reference to the Waku context, that will be needed in
                     the subsequent Waku API calls.
                     On error, returns a NULL, and the detail will be given in the `init_callback`.

  subscribe:
    description: Subscribes the waku instance to a content topic. On success, the waku instance
                 will start receiving messages containing the given content topic.
    parameters:
       - name: ctx
         type: pointer
         description: Reference to the Waku context, created by the init function.
       - name: contentTopic
         type: string
         description: Represents the topic of interest.
       - name: callback
         type: WakuCallback
         description: Gives feedback in case of success/failure when invoking the function.
       - name: msgEventCallback
         type: WakuCallback
         description: Invoked whenever a message containing the content topic of interest
                      is received.

  unsubscribe:
    description: Unsubscribes the waku instance from a certain content topic. On success, the waku
                 instance will stop receiving messages containing the given content topic.
    parameters:
       - name: ctx
         type: pointer
         description: Reference to the Waku context, created by the init function.
       - name: contentTopic
         type: string
         description: Represents the topic of interest.
       - name: callback
         type: WakuCallback
         description: Gives feedback in case of success/failure when invoking the function.
       - name: msgEventCallback
         type: WakuCallback
         description: Invoked whenever a message containing the content topic of interest
                      is received.
  
```

#### Functionality / Additional Information / Specific Behaviours

If the waku is operating in `client` mode, it MUST:

- Use [LIGHTPUSH](/standards/core/lightpush.md) to send messages
- Use [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) to receive messages
- Use [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/f08de108457eed828dadbd36339433c586701267/waku/standards/core/34/peer-exchange.md#abstract) to discover peers
- Use [STORE](../standards/core/store.md) as per [WAKU-P2P-RELIABILITY]()

If the waku is configured in `server` mode, it MUST:

- Use [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) protocol.
- Host endpoints for [LIGHTPUSH](../standards/core/lightpush.md) and [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md).
- Serve the [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/f08de108457eed828dadbd36339433c586701267/waku/standards/core/34/peer-exchange.md#abstract) protocol.

`client` SHOULD be used if in resource restricted environment,
whereas `server` SHOULD be used if node has no hardware and bandwidth restrictions.

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
