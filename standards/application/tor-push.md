---
title: TOR-PUSH
name: Waku v2 Tor Push
category: Best Current Practice
tags: [waku/application]
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

## Abstract

This document extends the [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md), specifying Waku Tor Push,
which allows nodes to push messages via Tor into the Waku relay network.

Waku Tor Push builds on [GOSSIPSUB-TOR-PUSH](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/gossipsub-tor-push.md).

**Protocol identifier**: /vac/waku/relay/2.0.0

Note: Waku Tor Push does not have a dedicated protocol identifier.
It uses the same identifier as Waku relay.
This allows Waku relay nodes that are oblivious to Tor Push to process messages received via Tor Push.

## Functional Operation

In its current version, Waku Tor Push corresponds to [GOSSIPSUB-TOR-PUSH](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/gossipsub-tor-push.md)
applied to [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md),
instead of [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md).

## Security/Privacy Considerations

see [GOSSIPSUB-TOR-PUSH](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/gossipsub-tor-push.md)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md)
- [GOSSIPSUB-TOR-PUSH](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/gossipsub-tor-push.md)
- [Tor](https://www.torproject.org/)
