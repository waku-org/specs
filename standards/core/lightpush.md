---
slug: 19
title: 19/WAKU2-LIGHTPUSH
name: Waku v2 Light Push
status: draft
editor: Zoltan Nagy <zoltan@status.im> 
contributors: 
  - Hanno Cornelius <hanno@status.im>
  - Daniel Kaiser <danielkaiser@status.im>
  - Oskar Thorén <oskarth@titanproxy.com>
---

previous version: `/vac/waku/lightpush/2.0.0-beta1` [19/WAKU2-LIGHTPUSH](https://rfc.vac.dev/waku/standards/core/19/lightpush)

---
**Protocol identifier**: `/vac/waku/lightpush/2.0.0-beta2`

## Motivation and Goals

Light nodes with short connection windows and limited bandwidth wish to publish messages into the Waku network.
Additionally, there is sometimes a need for confirmation that a message has been received "by the network"
(here, at least one node).

`19/WAKU2-LIGHTPUSH` is a request/response protocol for this.

## Payloads

```protobuf
syntax = "proto3";

message LightPushRequest {
    enum RequestType { // Future extension point
        RELAY = 0;
    }
    
    string request_id = 1;
    RequestType kind = 10;  // default is RELAY, placeholder for future extensions
    string pubsub_topic = 20;
    WakuMessage message = 21;
}

message LightPushResponse {
    enum Status {
        SUCCESS = 0;
        BAD_REQUEST = 400;
        NO_PEERS_TO_RELAY = 404;
        PAYLOAD_TOO_LARGE = 413;
        UNSUPPORTED_TOPIC = 415;
        TOO_MANY_REQUESTS = 429;
        INTERNAL_SERVER_ERROR = 500;
        SERVICE_UNAVAILABLE = 503;    
    }

    string request_id = 1;
    Status status_code = 10;
    optional string status_desc = 11;
    uint32 relay_peer_count = 12;
}
```

### Message Relaying

Nodes that respond to `LightPushRequest` MUST either
relay the encapsulated message via [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay) protocol on the specified `pubsub_topic`,
or forward the `LightPushRequest` via 19/LIGHTPUSH on a [WAKU2-DANDELION](https://github.com/waku-org/specs/blob/waku-RFC/standards/application/dandelion.md) stem.
If they are unable to do so for some reason, they SHOULD return an error code in `LightPushResponse`.

## Security Considerations

Since this can introduce an amplification factor, it is RECOMMENDED for the node relaying to the rest of the network to take extra precautions.
This can be done by rate limiting via [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay).

Note that the above is currently not fully implemented.

## Future work

- Add support attaching RLN proof for the message requested to be relayed.
- Add support for other request types.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

* [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay)
* [WAKU2-DANDELION](https://github.com/waku-org/specs/blob/waku-RFC/standards/application/dandelion.md)
* [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay)
