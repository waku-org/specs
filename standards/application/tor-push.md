---
title: TOR-PUSH
name: Waku v2 Tor Push
category: Best Current Practice
tags: [waku/application]
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

## Abstract

This document extends the [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay), specifying Waku Tor Push,
which allows nodes to push messages via Tor into the Waku relay network.

Waku Tor Push builds on [GOSSIPSUB-TOR-PUSH](https://rfc.vac.dev/vac/raw/gossipsub-tor-push).

**Protocol identifier**: /vac/waku/relay/2.0.0

Note: Waku Tor Push does not have a dedicated protocol identifier.
It uses the same identifier as Waku relay.
This allows Waku relay nodes that are oblivious to Tor Push to process messages received via Tor Push.

## Functional Operation

In its current version, Waku Tor Push corresponds to [GOSSIPSUB-TOR-PUSH](https://rfc.vac.dev/vac/raw/gossipsub-tor-push)
applied to [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay),
instead of [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md).

## Security/Privacy Considerations

see [GOSSIPSUB-TOR-PUSH](https://rfc.vac.dev/vac/raw/gossipsub-tor-push)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay)
- [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md)
- [GOSSIPSUB-TOR-PUSH](https://rfc.vac.dev/vac/raw/gossipsub-tor-push)
- [Tor](https://www.torproject.org/)
