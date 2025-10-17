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
    * [SDS integration](#sds-integration)
    * [Message segmentation](#message-segmentation)
    * [Rate limit management](#rate-limit-management)
  * [The Reliable Channel API](#the-reliable-channel-api)
    * [Create reliable channel](#create-reliable-channel)
      * [Type definitions](#type-definitions)
      * [Function definitions](#function-definitions)
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
    * [Synchronisation messages](#synchronisation-messages)
    * [Query on connect](#query-on-connect)
    * [Performance considerations](#performance-considerations)
    * [Ephemeral messages](#ephemeral-messages)
    * [Message segmentation](#message-segmentation-1)
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
- [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) (Scalable Data Sync) for causal ordering and acknowledgements
- Message segmentation for handling large payloads
- Rate limit management for [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md) compliance

The Reliable Channel API ensures that:

- All messages sent in a channel are eventually received by all participants
- Senders are notified when messages are acknowledged by other participants
- Missing messages are automatically detected and retrieved
- Message delivery is retried until acknowledged or maximum retry attempts are reached
- Messages are causally ordered using Lamport timestamps
- Large messages can be segmented to fit transport constraints
- Messages are queued or dropped when the underlying routing transport has a rate limit and said limit is being reached

## Motivation

While protocols like [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) provide the mechanisms for achieving reliability
(causal ordering, acknowledgements, missing message detection),
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
  - Message segmentation for large payloads
  - Rate limit compliance when using [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)

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
│  Message Segmentation                   │ ← Large message splitting/reassembly
├─────────────────────────────────────────┤
│  Rate Limit Manager                     │ ← WAKU2-RLN-RELAY compliance & pacing
├─────────────────────────────────────────┤
│  SDS (Scalable Data Sync)               │ ← Causal ordering & acknowledgements
├─────────────────────────────────────────┤
│  WAKU-API (LightPush/Filter/Store)      │ ← Message transport layer
└─────────────────────────────────────────┘
```

### SDS integration

The Reliable Channel wraps the [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) `MessageChannel`, which provides:

- **Causal ordering**: Using Lamport timestamps to establish message order
- **Acknowledgments**: Via causal history (definitive) and bloom filters (probabilistic)
- **Missing message detection**: By tracking gaps in causal history
- **Buffering**: For unacknowledged outgoing messages and incoming messages with unmet dependencies

The Reliable Channel handles the integration between SDS and Waku protocols:
- Wrapping user payloads in SDS messages before encoding
- Unwrapping SDS messages after decoding (extracting `content` field)
- Subscribing to messages via the Waku node's unified subscribe API
- Receiving messages via the node's message emitter (content-topic based)
- Scheduling SDS periodic tasks (sync, buffer sweeps, process tasks)
- Mapping SDS events to user-facing events
- Computing retrieval hints (Waku message hashes) for SDS messages

### Message segmentation

For messages exceeding safe payload limits:
- Messages SHOULD be split into segments of approximately 100 KB
- The 100 KB limit accounts for overhead from:
  - Encryption (message-encryption layer)
  - [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md) proof
  - SDS metadata (causal history, bloom filter, Lamport timestamp)
  - Protocol buffers encoding
- This ensures the final encoded Waku message stays well below the 150 KB routing layer limit
- Each segment SHOULD be tracked independently through SDS
- Segments SHOULD be reassembled before delivery to the application
- Partial message state SHOULD be managed to handle segment loss
- Segment order MUST be preserved during reassembly

TODO: refer to message segmentation spec

### Rate limit management

When using [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md):
- Regular messages and segments exceeding the rate limit SHOULD be queued
- Ephemeral messages SHOULD be dropped (not sent nor queued) when the rate limit is approached or reached
- "Approached" threshold SHOULD be configurable (e.g., drop ephemerals at 90% capacity)
- Rate limit errors SHOULD be surfaced through events
- When segmentation is enabled, each segment counts toward the rate limit independently
- Messages coming from a retry mechanism should be queued when the rate limit is approached.

TODO: refer to rate limit manager spec

Note: at a later stage, we may prefer for the Waku node to expose rate limit information (total rate limit, usage so far)
instead of having it a configurable on the reliable channel.

## The Reliable Channel API

```yaml
api_version: "0.0.1"
library_name: "waku-reliable-channel"
description: "Reliable Channel: an event-driven API for eventual message consistency in Waku channels."
```

### Create reliable channel

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
        description: "Whether to automatically do a store query after connection to store nodes. Note: this should be moved to the Waku API."
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
      timeout_for_lost_messages_ms:
        type: uint
        description: "The time in milliseconds after which a message dependency that could not be resolved is marked as irretrievable. Disabled if undefined or 0. Passed to the underlying SDS MessageChannel."
      possible_acks_threshold:
        type: uint
        description: "How many possible acknowledgements (bloom filter hits) does it take to consider it a definitive acknowledgement. Passed to the underlying SDS MessageChannel."

  ChannelId:
    type: string
    description: "An identifier for the channel. All participants of the channel MUST use the same id."

  SenderId:
    type: string
    description: "An identifier for the sender. SHOULD be unique per participant and persisted between sessions to ensure acknowledgements are only valid when originating from different senders."

  MessageId:
    type: string
    description: "A unique identifier for a logical message, derived from the message payload before segmentation."

  MessagePayload:
    type: array<byte>
    description: "The unwrapped message content (user payload extracted from SDS message)."
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
      - name: content_topic
        type: ContentTopic
        description: "The content topic to use for the channel."
      - name: options
        type: ReliableChannelOptions
        description: "Configuration options for the Reliable Channel."
    returns:
      type: result<ReliableChannel, error>
```

#### Extended definitions

**Default configuration values**: See the [SDS Implementation Suggestions](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md#sdk-usage-reliablechannel) section for recommended default values for `ReliableChannelOptions`.

**`channel_id` and `sender_id`**:

The `channel_id` MUST be the same for all participants in a channel.
The `sender_id` SHOULD be unique for each participant and SHOULD be persisted between sessions to ensure proper acknowledgement tracking.

**`content_topic`**:

A Reliable Channel uses a unique content topic. This ensure that all messages are retrievable.

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

  sendEphemeral:
    description: "Send an ephemeral message in the channel. Ephemeral messages are not tracked for acknowledgement, not included in causal history, and are dropped when rate limits are approached or reached."
    parameters:
      - name: message_payload
        type: array<byte>
        description: "The message content to send (before SDS wrapping)."
    returns:
      type: MessageId
      description: "A unique identifier for the message, used to track events (limited to `message-sent` or `ephemeral-message-dropped` event)."
```

### Event handling

The Reliable Channel uses an event-driven model to notify applications about message lifecycle events.

#### Type definitions

```yaml
types:
  ChunkInfo:
    type: object
    description: "Information about message segmentation for tracking partial progress."
    fields:
      chunks:
        type: array<uint>
        description: "Array of chunk indices (e.g., [0, 1, 3] means chunks 0, 1, and 3). For non-segmented messages, this is [0]."
      total_chunks:
        type: uint
        description: "Total number of chunks for this message. For non-segmented messages, this is 1. When chunks.length == total_chunks, all chunks are complete."

  ReliableChannelEvents:
    type: object
    description: "Events emitted by the Reliable Channel."
    events:
      sending-message:
        type: event<SendingMessageEvent>
        description: "Emitted when a message chunk is being sent over the wire. MAY be emitted multiple times if retry mechanism kicks in. For segmented messages, emitted once per chunk."

      message-sent:
        type: event<MessageSentEvent>
        description: "Emitted when a message chunk has been sent over the wire but has not been acknowledged yet. MAY be emitted multiple times if retry mechanism kicks in. For segmented messages, emitted once per chunk."

      message-possibly-acknowledged:
        type: event<PossibleAcknowledgment>
        description: "Emitted when a bloom filter indicates a message chunk was possibly received by another party. This is probabilistic. For segmented messages, emitted per chunk."

      message-acknowledged:
        type: event<MessageAcknowledgedEvent>
        description: "Emitted when a message chunk was fully acknowledged by other members of the channel (present in their causal history). For segmented messages, emitted per chunk as each is acknowledged."

      sending-message-irrecoverable-error:
        type: event<MessageError>
        description: "Emitted when a message chunk could not be sent due to a non-recoverable error. For segmented messages, emitted per chunk that fails."

      message-received:
        type: event<MessagePayload>
        description: "Emitted when a new complete message has been received and reassembled from another participant. The payload is the unwrapped user content (SDS content field). Only emitted once all chunks are received."

      irretrievable-message:
        type: event<HistoryEntry>
        description: "Emitted when the channel is aware of a missing message but failed to retrieve it successfully."

      ephemeral-message-dropped:
        type: event<EphemeralDropReason>
        description: "Emitted when an ephemeral message was dropped due to rate limit constraints."

  SendingMessageEvent:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices being sent. The chunks array contains all chunks sending so far."

  MessageSentEvent:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have been sent. The chunks array contains all chunks sent so far."

  MessageAcknowledgedEvent:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have been acknowledged. The chunks array contains all acknowledged chunks so far. When chunks.length == total_chunks, all chunks are acknowledged."

  PossibleAcknowledgment:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID that was possibly acknowledged."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have been possibly acknowledged (via bloom filters). The chunks array contains all possibly acknowledged chunks so far. Note: this may not be an ideal API to use, further revision may be needed."

  MessageError:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID that encountered an error."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have failed. The chunks array contains all failed chunks so far."
      error:
        type: error
        description: "The error that occurred."

  HistoryEntry:
    type: object
    description: "An entry in the message history that could not be retrieved."

  EphemeralDropReason:
    type: object
    fields:
      reason:
        type: string
        description: "The reason the ephemeral message was dropped (e.g., 'rate_limit_approached', 'rate_limit_reached')."
```

#### Extended definitions

**Event lifecycle**:

For each regular message sent via `send()`, the following event sequence is expected:

1. `sending-message`: Emitted each time chunks are sent, with cumulative array (MAY be emitted multiple times if retry mechanism kicks in)
   - Example progression for a 1-chunk message:
     - sendindg: `chunk_info={chunks: [0], total_chunks: 1}`
   - Example progression for a 5-chunk message:
     - First send: `chunk_info={chunks: [0], total_chunks: 5}`
     - Second send: `chunk_info={chunks: [0, 1], total_chunks: 5}`
     - Fifth send: `chunk_info={chunks: [0, 1, 2, 3, 4], total_chunks: 5}`
2. `message-sent`: Emitted with same cumulative pattern as `sending-message` (MAY be emitted multiple times if retry mechanism kicks in)
3. `message-acknowledged`: Emitted each time new chunks are acknowledged
   - `chunk_info.chunks` array grows as chunks are acknowledged
   - Example progression for a 1-chunk message:
     - Only ack: `chunk_info={chunks: [0], total_chunks: 1}` (complete when `chunks.length == total_chunks`)
   - Example progression for a 5-chunk message:
     - First ack: `chunk_info={chunks: [0], total_chunks: 5}`
     - Second ack: `chunk_info={chunks: [0, 1], total_chunks: 5}`
     - Third ack: `chunk_info={chunks: [0, 1, 3], total_chunks: 5}` (chunk 2 still pending)
     - Fourth ack: `chunk_info={chunks: [0, 1, 2, 3], total_chunks: 5}` (chunk 2 now received)
     - Final ack: `chunk_info={chunks: [0, 1, 2, 3, 4], total_chunks: 5}` (complete when `chunks.length == total_chunks`)
   - All events share the same `message_id`
   - Application can:
     - Check overall progress: `chunk_info.chunks.length / chunk_info.total_chunks` (e.g., 3/5 = 60%)
     - Check if complete: `chunk_info.chunks.length == chunk_info.total_chunks`
     - Track specific chunks: `chunk_info.chunks.includes(2)` to see if chunk 2 is done
     - Display which chunks remain: chunks not in the array

Alternatively, instead of `message-acknowledged`, the message lifecycle MAY end with:
   - `sending-message-irrecoverable-error`: Emitted if an unrecoverable error occurs

Optionally, before final acknowledgement:
   - `message-possibly-acknowledged`: (Probabilistic) MAY be emitted as bloom filter hits are detected

**For received messages**:
- `message-received`: Emitted **only once** after all chunks are received and reassembled. Note: we may want to provide progressive reception information.
- The payload is the complete-reassembled message
- No chunk information is provided (reassembly is transparent to the receiver)

For ephemeral messages sent via `sendEphemeral()`:
- Ephemeral messages are NEVER segmented (if too large, they are rejected)
- `message-sent`: Emitted once with `chunk_info={chunks: [0], total_chunks: 1}`
- `ephemeral-message-dropped`: Emitted if the message is dropped due to rate limit constraints
- `sending-message-irrecoverable-error`: Emitted if encoding, sending, or size check fails

**Irrecoverable errors**:

The following errors are considered irrecoverable and will trigger `sending-message-irrecoverable-error`:
- Encoding failed
- Empty payload
- Message size too large
- WAKU2-RLN-RELAY proof generation failed

When an irrecoverable error occurs, the retry mechanism SHOULD NOT attempt to resend the message.

### Channel lifecycle

#### Function definitions

```yaml
functions:
  start:
    description: "Start the Reliable Channel. Sets up event listeners, begins sync loop, starts missing message retrieval, and subscribes to messages via the Waku node."
    returns:
      type: void

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
   User payload → SDS ContentMessage (encode) → Waku Message → Network
   ```

2. **Message unwrapping**: Received Waku messages MUST be unwrapped to extract user payloads:
   ```
   Network → Waku Message → SDS Message (decode) → User payload (content field)
   ```
   - The Reliable Channel receives raw `Uint8Array` payloads from the node's message emitter
   - SDS decoding extracts the message structure
   - Only the `content` field is emitted to the application via `message-received` event

3. **SDS configuration**: Implementations SHOULD configure the SDS MessageChannel with:
   - `causalHistorySize`: Number of recent message IDs in causal history (default: 200)
   - `bloomFilterSize`: Bloom filter capacity for probabilistic ACKs (default: 10,000 messages)
   - `bloomFilterErrorRate`: False positive rate (default: 0.001)

4. **Subscription and reception**:
   - Call `node.subscribe([contentTopic])` to subscribe via the Waku node's unified API
   - Listen to `node.messageEmitter` on the content topic for incoming messages
   - Process raw `Uint8Array` payloads through SDS decoding
   - Extract and emit the `content` field to the application

5. **Retrieval hint computation**:
   - Compute Waku message hash from the SDS-wrapped payload before sending
   - Include hash in SDS `HistoryEntry` as `retrievalHint`
   - Use hash for targeted store queries when retrieving missing messages

6. **Task scheduling**: Implementations MUST periodically call SDS methods:
   - `processTasks()`: Process queued send/receive operations
   - `sweepIncomingBuffer()`: Deliver messages with met dependencies
   - `sweepOutgoingBuffer()`: Identify messages for retry

7. **Event mapping**: SDS events SHOULD be mapped to Reliable Channel events:
   - `OutMessageSent` → `message-sent`
   - `OutMessageAcknowledged` → `message-acknowledged`
   - `OutMessagePossiblyAcknowledged` → `message-possibly-acknowledged`
   - `InMessageReceived` → (internal, triggers sync restart)
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

### Synchronisation messages

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

### Ephemeral messages

Ephemeral messages are defined in the [SDS specification](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md#ephemeral-messages) as short-lived messages for which no synchronisation or reliability is required.

**Implementation notes**:

1. **SDS integration**:
   - Send ephemeral messages with `lamport_timestamp`, `causal_history`, and `bloom_filter` unset
   - Do NOT add to unacknowledged outgoing buffer after broadcast
   - Do NOT include in causal history or bloom filters
   - Do NOT add to local log

2. **Rate limit awareness**:
   - Before sending an ephemeral message, check rate limit utilization
   - If utilization >= threshold (default: 90%), drop the message and emit `ephemeral-message-dropped`
   - This ensures reliable messages are never blocked by ephemeral traffic

3. **Use cases**:
   - Typing indicators
   - Presence updates
   - Real-time status updates
   - Other transient UI state that doesn't require guaranteed delivery

4. **Receiving ephemeral messages**:
   - Deliver immediately without buffering for causal dependencies
   - Emit `message-received` event (same as regular messages)
   - Do NOT add to local log or acknowledge

**Reference**: SDS specification section on [Ephemeral Messages](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md#ephemeral-messages)

### Message segmentation

To handle large messages while accounting for protocol overhead, implementations SHOULD:

1. **Segment size calculation**:
   - Target segment size: ~100 KB (102,400 bytes) of user payload
   - This accounts for overhead that will be added:
     - Encryption: Variable depending on encryption scheme (e.g., ~48 bytes for ECIES)
     - SDS metadata: ~12.8 KB with default causal history (200 × 64 bytes)
     - WAKU2-RLN-RELAY proof: ~128 bytes
     - Protobuf encoding overhead: ~few hundred bytes
   - Final Waku message stays well under 150 KB routing layer limit

2. **Message ID and chunk tracking**:
   - Compute a single logical `message_id` from the **complete** user payload (before segmentation)
   - All chunks of the same message share this `message_id`
   - Each chunk has its own SDS message ID for tracking in causal history
   - Chunk info (`chunk_index`, `total_chunks`) is included in all events

3. **Segmentation strategy**:
   - Split large user payloads into ~100 KB chunks
   - Wrap each chunk in a separate SDS `ContentMessage`
   - Include segmentation metadata in each SDS message (chunk index, total chunks, logical message ID)
   - Each chunk is sent and tracked independently through SDS

4. **Event emission**:
   - `sending-message`: Emitted each time chunks are sent with cumulative `chunk_info`
   - `message-sent`: Emitted each time chunks are sent with cumulative `chunk_info`
   - `message-acknowledged`: Emitted each time new chunks are acknowledged
     - `chunk_info.chunks` contains ALL acknowledged chunks so far (cumulative)
     - Example progression: `{chunks: [0], total_chunks: 5}` → `{chunks: [0, 1], total_chunks: 5}` → `{chunks: [0, 1, 3], total_chunks: 5}` → `{chunks: [0, 1, 2, 3, 4], total_chunks: 5}`
     - Application can show: `${chunk_info.chunks.length}/${chunk_info.total_chunks} chunks sent (60%)`
     - Check if complete: `chunk_info.chunks.length == chunk_info.total_chunks`
     - Track individual chunks: `!chunk_info.chunks.includes(2)` to show "chunk 2 pending"
   - All events for the same logical message share the same `message_id`

5. **Reassembly (receiving)**:
   - Buffer received chunks keyed by logical message ID
   - Track which chunks have been received (by chunk index)
   - Verify chunk count matches expected total from metadata
   - Reassemble in sequence order when all chunks present
   - Emit `message-received` **only once** with complete payload
   - Handle partial message timeout (mark as irretrievable after threshold)

6. **Interaction with retries and rate limits**:
   - Each chunk is retried independently if not acknowledged
   - When using WAKU2-RLN-RELAY, each chunk consumes one message slot per epoch
   - Large messages may take multiple epochs to send completely
   - Partial acknowledgements allow applications to show progress to users

## Security/Privacy Considerations

1. **Encryption**: All participants in a Reliable Channel MUST be able to decrypt messages.
   Implementations SHOULD use the same encryption layer (encoder/decoder) for all messages.

2. **Sender identity**: The `sender_id` is used to differentiate acknowledgements.
   Implementations SHOULD ensure that acknowledgements are only considered valid when they originate from a different sender.

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
