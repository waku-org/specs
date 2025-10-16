---
title: RELIABLE-CHANNEL-API
name: Reliable Channel API
category: Standards Track
status: raw
tags: [reliability, application, api]
editor: Franck Royer <franck@status.im>
contributors:
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
  * [Architecture](#architecture)
    * [SDS Integration](#sds-integration)
    * [Message Segmentation (Future)](#message-segmentation-future)
    * [Rate Limit Management (Future)](#rate-limit-management-future)
  * [The Reliable Channel API](#the-reliable-channel-api)
    * [Create Reliable Channel](#create-reliable-channel)
      * [Type definitions](#type-definitions)
      * [Function definitions](#function-definitions)
      * [Predefined values](#predefined-values)
      * [Extended definitions](#extended-definitions)
    * [Send messages](#send-messages)
      * [Function definitions](#function-definitions-1)
    * [Event handling](#event-handling)
      * [Type definitions](#type-definitions-1)
      * [Extended definitions](#extended-definitions-1)
    * [Channel lifecycle](#channel-lifecycle)
      * [Function definitions](#function-definitions-2)
  * [Implementation Suggestions](#implementation-suggestions)
    * [SDS MessageChannel integration](#sds-messagechannel-integration)
    * [Message retries](#message-retries)
    * [Missing message retrieval](#missing-message-retrieval)
    * [Synchronization messages](#synchronization-messages)
    * [Query on connect](#query-on-connect)
    * [Performance considerations](#performance-considerations)
  * [Security/Privacy Considerations](#securityprivacy-considerations)
  * [Copyright](#copyright)
<!-- TOC -->

## Abstract

This document specifies an Application Programming Interface (API) for Reliable Channel,
a high-level abstraction that provides eventual message consistency guarantees for all participants in a channel,
as well as message segmentation and rate limit management when using an underlying rate limited delivery protocol
with message size restrictions such as [WAKU2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md).

The Reliable Channel is built on top of:
- [WAKU-API](/standards/application/waku-api.md) for Waku protocol integration
- [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) (Scalable Data Sync) for causal ordering and acknowledgments
- Message segmentation for handling large payloads (TBD)
- Rate limit management for RLN compliance (TBD)

The Reliable Channel API ensures that:

- All messages sent in a channel are eventually received by all participants
- Senders are notified when messages are acknowledged by other participants
- Missing messages are automatically detected and retrieved
- Message delivery is retried until acknowledged or maximum retry attempts are reached
- Messages are causally ordered using Lamport timestamps
- Large messages can be segmented to fit transport constraints (TBD)

## Motivation

While protocols like [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) provide the mechanisms for achieving reliability
(causal ordering, acknowledgments, missing message detection),
and [WAKU-API](/standards/application/waku-api.md) provides the transport layer,
there is a need for an opinionated, high-level API that makes these capabilities accessible and easy to use.

The Reliable Channel API provides this accessibility by:
- **Simplifying integration**: Wraps SDS, Waku protocols, and other components (segmentation, rate limiting) behind a single, cohesive interface
- **Providing sane defaults**: Pre-configures SDS parameters, retry strategies, and sync intervals for common use cases
- **Event-driven model**: Exposes message lifecycle through intuitive events rather than requiring manual polling of SDS state
- **Automatic task scheduling**: Handles the periodic execution of SDS tasks (sync, buffer sweeps) internally
- **Abstracting complexity**: Hides the details of:
  - SDS message wrapping/unwrapping
  - Store queries for missing messages
  - Query-on-connect behavior
  - Message segmentation for large payloads (TBD)
  - Rate limit compliance when using RLN (TBD)

The goal is to enable application developers to achieve end-to-end reliability
with minimal configuration and without deep knowledge of the underlying protocols.
This follows the same philosophy as [WAKU-API](/standards/application/waku-api.md):
providing an opinionated, accessible interface to powerful but complex underlying mechanisms.

## Syntax

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## API design

### IDL

This specification uses the same custom Interface Definition Language (IDL) in YAML as defined in [WAKU-API](/standards/application/waku-api.md).

### Primitive types and general guidelines

The primitive types and general guidelines are the same as defined in [WAKU-API](/standards/application/waku-api.md#primitive-types-and-general-guidelines).

## Architecture

The Reliable Channel is a layered architecture that combines multiple components:

```
┌─────────────────────────────────────────┐
│      Reliable Channel API               │ ← Application-facing event-driven API
├─────────────────────────────────────────┤
│  Message Segmentation (future)          │ ← Large message splitting/reassembly
├─────────────────────────────────────────┤
│  Rate Limit Manager (future)            │ ← RLN compliance and pacing
├─────────────────────────────────────────┤
│  SDS (Scalable Data Sync)               │ ← Causal ordering & acknowledgments
├─────────────────────────────────────────┤
│  WAKU-API (LightPush/Filter/Store)      │ ← Message transport layer
└─────────────────────────────────────────┘
```

### SDS Integration

The Reliable Channel wraps the [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) `MessageChannel` to provide:

- **Causal ordering**: Using Lamport timestamps to establish message order
- **Acknowledgments**: Via causal history (definitive) and bloom filters (probabilistic)
- **Missing message detection**: By tracking gaps in causal history
- **Buffering**: For unacknowledged outgoing messages and incoming messages with unmet dependencies

The Reliable Channel handles the integration between SDS and Waku protocols:
- Wrapping user payloads in SDS messages before encoding
- Unwrapping SDS messages after decoding
- Scheduling SDS periodic tasks (sync, buffer sweeps, process tasks)
- Mapping SDS events to user-facing events

### Message Segmentation (Future)

For messages exceeding transport limits (e.g., 150 KiB for Waku with RLN):
- Messages SHOULD be split into multiple segments
- Each segment SHOULD be tracked independently through SDS
- Segments SHOULD be reassembled before delivery to the application
- Partial message state SHOULD be managed to handle segment loss

### Rate Limit Management (Future)

When using [RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md):
- The Reliable Channel SHOULD pace message sending to comply with rate limits
- Messages exceeding the rate limit SHOULD be queued
- RLN proofs SHOULD be generated for each message/segment
- Rate limit errors SHOULD be surfaced through events

## The Reliable Channel API

```yaml
api_version: "0.0.1"
library_name: "waku-reliable-channel"
description: "Reliable Channel: an event-driven API for eventual message consistency in Waku channels."
```

### Create Reliable Channel

#### Type definitions

```yaml
types:
  ReliableChannel:
    type: object
    description: "A Reliable Channel instance that provides eventual consistency guarantees."

  ReliableChannelOptions:
    type: object
    fields:
      sync_min_interval_ms:
        type: uint
        default: 30000
        description: "The minimum interval between 2 sync messages in the channel (in milliseconds). This is shared responsibility between channel participants. Set to 0 to disable automatic sync messages."
      retry_interval_ms:
        type: uint
        default: 30000
        description: "How long to wait before re-sending a message that has not been acknowledged (in milliseconds)."
      max_retry_attempts:
        type: uint
        default: 10
        description: "How many times to attempt resending messages that were not acknowledged."
      retrieve_frequency_ms:
        type: uint
        default: 10000
        description: "How often store queries are done to retrieve missing messages (in milliseconds)."
      sweep_in_buf_interval_ms:
        type: uint
        default: 5000
        description: "How often the SDS message channel incoming buffer is swept (in milliseconds)."
      query_on_connect:
        type: bool
        default: true
        description: "Whether to automatically do a store query after connection to store nodes."
      auto_start:
        type: bool
        default: true
        description: "Whether to automatically start the message channel."
      process_task_min_elapse_ms:
        type: uint
        default: 1000
        description: "The minimum elapsed time between calling the underlying channel process task for incoming messages. This prevents overload when processing many messages."
      causal_history_size:
        type: uint
        description: "The number of recent messages to include in causal history. Passed to the underlying SDS MessageChannel."
      bloom_filter_size:
        type: uint
        description: "The size of the bloom filter for probabilistic acknowledgments. Passed to the underlying SDS MessageChannel."

  ChannelId:
    type: string
    description: "An identifier for the channel. All participants of the channel MUST use the same id."

  SenderId:
    type: string
    description: "An identifier for the sender. SHOULD be unique per participant and persisted between sessions to ensure acknowledgements are only valid when originating from different senders."

  MessageId:
    type: string
    description: "A unique identifier for a message, derived from the message payload."

  DecodedMessage:
    type: object
    description: "A decoded Waku message with unwrapped payload."
    fields:
      payload:
        type: array<byte>
        description: "The unwrapped message content."
      timestamp:
        type: uint
        description: "The message timestamp."
      content_topic:
        type: string
        description: "The content topic of the message."
      pubsub_topic:
        type: string
        description: "The pubsub topic of the message."
      hash:
        type: array<byte>
        description: "The message hash."
      ephemeral:
        type: bool
        description: "Whether the message is ephemeral."
      meta:
        type: array<byte>
        description: "Optional metadata."
```

#### Function definitions

```yaml
functions:
  createReliableChannel:
    description: "Create a new Reliable Channel instance. All participants in the channel MUST be able to decrypt messages and MUST subscribe to the same content topic(s)."
    parameters:
      - name: waku_node
        type: WakuNode
        description: "The Waku node instance to use for sending and receiving messages."
      - name: channel_id
        type: ChannelId
        description: "An identifier for the channel. All participants MUST use the same id."
      - name: sender_id
        type: SenderId
        description: "An identifier for this sender. SHOULD be unique and persisted between sessions."
      - name: encoder
        type: Encoder
        description: "The encoder for messages. All messages in the channel use the same encryption layer."
      - name: decoder
        type: Decoder
        description: "The decoder for messages. All messages in the channel use the same encryption layer."
      - name: options
        type: ReliableChannelOptions
        description: "Configuration options for the Reliable Channel."
    returns:
      type: result<ReliableChannel, error>
```

#### Predefined values

```yaml
values:
  DefaultReliableChannelOptions:
    type: ReliableChannelOptions
    fields:
      sync_min_interval_ms: 30000
      retry_interval_ms: 30000
      max_retry_attempts: 10
      retrieve_frequency_ms: 10000
      sweep_in_buf_interval_ms: 5000
      query_on_connect: true
      auto_start: true
      process_task_min_elapse_ms: 1000
```

#### Extended definitions

**`channel_id` and `sender_id`**:

The `channel_id` MUST be the same for all participants in a channel.
The `sender_id` SHOULD be unique for each participant and SHOULD be persisted between sessions to ensure proper acknowledgment tracking.

**`encoder` and `decoder`**:

A Reliable Channel operates within a singular encryption layer.
All messages sent and received in the channel MUST use the same encoder and decoder.
This ensures that:
- All participants can decrypt all messages
- Messages are sent to the same content topic(s)

**`options.auto_start`**:

If set to `true` (default), the Reliable Channel SHOULD automatically call `start()` during creation.
If set to `false`, the application MUST call `start()` before the channel will process messages.

**`options.query_on_connect`**:

If set to `true` (default) and the Waku node has store capability,
the Reliable Channel SHOULD automatically query the store for missing messages when connecting to store nodes.
This helps ensure message consistency when participants come online after being offline.

### Send messages

#### Function definitions

```yaml
functions:
  send:
    description: "Send a message in the channel. The message will be retried if not acknowledged by other participants."
    parameters:
      - name: message_payload
        type: array<byte>
        description: "The message content to send (before SDS wrapping)."
    returns:
      type: MessageId
      description: "A unique identifier for the message, used to track events."

  getMessageId:
    description: "Get the message ID for a given payload. Used to track message events before sending."
    static: true
    parameters:
      - name: message_payload
        type: array<byte>
        description: "The message content (before SDS wrapping)."
    returns:
      type: MessageId
      description: "The unique identifier that will be used for this message."
```

### Event handling

The Reliable Channel uses an event-driven model to notify applications about message lifecycle events.

#### Type definitions

```yaml
types:
  ReliableChannelEvents:
    type: object
    description: "Events emitted by the Reliable Channel."
    events:
      sending-message:
        type: event<MessageId>
        description: "Emitted when a message is being sent over the wire. MAY be emitted multiple times if retry mechanism kicks in."

      message-sent:
        type: event<MessageId>
        description: "Emitted when a message has been sent over the wire but has not been acknowledged yet. MAY be emitted multiple times if retry mechanism kicks in."

      message-possibly-acknowledged:
        type: event<PossibleAcknowledgment>
        description: "Emitted when a bloom filter indicates the message was possibly received by another party. This is probabilistic. Retry mechanism will wait longer before trying again."

      message-acknowledged:
        type: event<MessageId>
        description: "Emitted when a message was fully acknowledged by other members of the channel (present in their causal history)."

      sending-message-irrecoverable-error:
        type: event<MessageError>
        description: "Emitted when a message could not be sent due to a non-recoverable error (likely an internal error)."

      message-received:
        type: event<DecodedMessage>
        description: "Emitted when a new message has been received from another participant."

      irretrievable-message:
        type: event<HistoryEntry>
        description: "Emitted when the channel is aware of a missing message but failed to retrieve it successfully."

  PossibleAcknowledgment:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The message ID that was possibly acknowledged."
      possible_ack_count:
        type: uint
        description: "The number of possible acknowledgments detected."

  MessageError:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The message ID that encountered an error."
      error:
        type: error
        description: "The error that occurred."

  HistoryEntry:
    type: object
    description: "An entry in the message history that could not be retrieved."
```

#### Extended definitions

**Event lifecycle**:

For each message sent, the following event sequence is expected:

1. `sending-message`: Emitted when the message encoding and sending begins
2. `message-sent`: Emitted when the message has been successfully sent over the network
3. One of:
   - `message-possibly-acknowledged`: (Optional, probabilistic) Emitted when bloom filters suggest acknowledgment
   - `message-acknowledged`: Emitted when causal history confirms acknowledgment
   - `sending-message-irrecoverable-error`: Emitted if an unrecoverable error occurs

Events 1-2 MAY be emitted multiple times if the retry mechanism is activated due to lack of acknowledgment.

**Irrecoverable errors**:

The following errors are considered irrecoverable and will trigger `sending-message-irrecoverable-error`:
- Encoding failed
- Empty payload
- Message size too large
- RLN proof generation failed

When an irrecoverable error occurs, the retry mechanism SHOULD NOT attempt to resend the message.

### Channel lifecycle

#### Function definitions

```yaml
functions:
  start:
    description: "Start the Reliable Channel. Sets up event listeners, begins sync loop, starts missing message retrieval, and subscribes to messages."
    returns:
      type: result<bool, error>
      description: "True if successfully started, error otherwise."

  stop:
    description: "Stop the Reliable Channel. Stops sync loop, missing message retrieval, and clears intervals."
    returns:
      type: void

  isStarted:
    description: "Check if the Reliable Channel is currently started."
    returns:
      type: bool
      description: "True if the channel is started, false otherwise."
```

## Implementation Suggestions

This section provides practical implementation guidance based on the [js-waku](https://github.com/waku-org/js-waku) implementation.

### SDS MessageChannel integration

The Reliable Channel MUST use the [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) `MessageChannel` as its core reliability mechanism.

**Reference**: `js-waku/packages/sds/src/message_channel/message_channel.ts:1`

**Key integration points**:

1. **Message wrapping**: User payloads MUST be wrapped in SDS `ContentMessage` before sending:
   ```
   User payload → SDS ContentMessage → Waku Message → Network
   ```

2. **Message unwrapping**: Received Waku messages MUST be unwrapped to extract user payloads:
   ```
   Network → Waku Message → SDS ContentMessage → User payload
   ```

3. **SDS configuration**: Implementations SHOULD configure the SDS MessageChannel with:
   - `causalHistorySize`: Number of recent message IDs in causal history (default: 200)
   - `bloomFilterSize`: Bloom filter capacity for probabilistic ACKs (default: 10,000 messages)
   - `bloomFilterErrorRate`: False positive rate (default: 0.001)

4. **Task scheduling**: Implementations MUST periodically call SDS methods:
   - `processTasks()`: Process queued send/receive operations
   - `sweepIncomingBuffer()`: Deliver messages with met dependencies
   - `sweepOutgoingBuffer()`: Identify messages for retry

5. **Event mapping**: SDS events SHOULD be mapped to Reliable Channel events:
   - `OutMessageSent` → `message-sent`
   - `OutMessageAcknowledged` → `message-acknowledged`
   - `OutMessagePossiblyAcknowledged` → `message-possibly-acknowledged`
   - `InMessageReceived` → `message-received`
   - `InMessageMissing` → Missing message retrieval trigger

**Default SDS configuration values** (from js-waku):
- Bloom filter capacity: 10,000 messages
- Bloom filter error rate: 0.001 (0.1% false positive rate)
- Causal history size: 200 message IDs (≈12.8 KB overhead per message)
- Possible ACKs threshold: 2 bloom filter hits before considering acknowledged

### Message retries

The retry mechanism SHOULD use a simple fixed-interval retry strategy:

- When a message is sent, start a retry timer
- Every `retry_interval_ms`, attempt to resend the message
- Stop retrying when:
  - The message is acknowledged (via causal history)
  - `max_retry_attempts` is reached
  - An irrecoverable error occurs

**Reference**: `js-waku/packages/sdk/src/reliable_channel/retry_manager.ts:1`

**Implementation notes**:

- Retry intervals SHOULD be configurable (default: 30 seconds)
- Maximum retry attempts SHOULD be configurable (default: 10 attempts)
- Future implementations MAY implement exponential back-off strategies
- Implementations SHOULD NOT retry sending if the node is known to be offline

### Missing message retrieval

The Reliable Channel SHOULD implement automatic detection and retrieval of missing messages:

1. **Detection**: When processing incoming messages, the SDS layer detects gaps in causal history
2. **Tracking**: Missing messages are tracked with their message IDs and retrieval hints (Waku message hashes)
3. **Retrieval**: Periodic store queries retrieve missing messages using retrieval hints
4. **Processing**: Retrieved messages are processed through the normal message pipeline

**Reference**: `js-waku/packages/sdk/src/reliable_channel/missing_message_retriever.ts:1`

**Implementation notes**:

- Missing message checks SHOULD run periodically (default: every 10 seconds)
- Store queries SHOULD use message hashes for targeted retrieval
- Retrieved messages SHOULD be removed from the missing messages list once received
- If a message cannot be retrieved, implementations MAY emit an `irretrievable-message` event

### Synchronization messages

Sync messages are empty messages that carry only causal history and bloom filter information.
They serve to:
- Acknowledge received messages without sending new content
- Keep the channel active
- Allow participants to learn about each other's message state

**Implementation notes**:

- Sync messages SHOULD be sent periodically with randomized delays
- The sync interval is shared responsibility: `sync_interval = random() * sync_min_interval_ms`
- When a content message is received, sync SHOULD be scheduled sooner (multiplier: 0.5)
- When a content message is sent, sync SHOULD be rescheduled at normal interval (multiplier: 1.0)
- When a sync message is received, sync SHOULD be rescheduled at normal interval
- After failing to send a sync, retry SHOULD use a longer interval (multiplier: 2.0)

**Reference**: `js-waku/packages/sdk/src/reliable_channel/reliable_channel.ts:515-529`

### Query on connect

When enabled, the Reliable Channel SHOULD automatically query the store when connecting to store nodes.
This helps participants catch up on missed messages after being offline.

**Implementation notes**:

- Query on connect SHOULD be triggered when:
  - A store node becomes available
  - The node reconnects after being offline (health status changes)
  - A configurable time threshold has elapsed since the last query (default: 5 minutes)
- Queries SHOULD stop when finding a message with causal history from the same channel
- Queries SHOULD continue if all retrieved messages are from different channels

**Reference**: `js-waku/packages/sdk/src/reliable_channel/reliable_channel.ts:183-196`

### Performance considerations

To avoid overload when processing many messages:

1. **Throttled processing**: Don't call the SDS process task more frequently than `process_task_min_elapse_ms` (default: 1 second)
2. **Batched sweeping**: Sweep the incoming buffer at regular intervals rather than per-message (default: every 5 seconds)
3. **Lazy task execution**: Queue process tasks with a minimum elapsed time between executions

**Reference**: `js-waku/packages/sdk/src/reliable_channel/reliable_channel.ts:460-473`

## Security/Privacy Considerations

1. **Encryption**: All participants in a Reliable Channel MUST be able to decrypt messages.
   Implementations SHOULD use the same encryption layer (encoder/decoder) for all messages.

2. **Sender identity**: The `sender_id` is used to differentiate acknowledgments.
   Implementations SHOULD ensure that acknowledgments are only considered valid when they originate from a different sender.

3. **Channel isolation**: Messages in different channels are isolated.
   A participant SHOULD only process messages that match their channel ID.

4. **Message ordering**: While the Reliable Channel ensures eventual consistency,
   it does not guarantee strict message ordering across participants.

5. **Resource exhaustion**: Implementations SHOULD implement limits on:
   - Number of missing messages tracked
   - Number of active retry attempts
   - Frequency of store queries

6. **Privacy**: Store queries reveal message interest patterns.
   Implementations MAY consider privacy-preserving retrieval strategies in the future.

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
