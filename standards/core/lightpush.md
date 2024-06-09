---
title: WAKU2-LIGHTPUSH
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
**Protocol identifier**: `/vac/waku/lightpush/3.0.0`

## Motivation and Goals

Light nodes with short connection windows and limited bandwidth wish to push messages to other nodes in the Waku network to request message services.
A common use case is to request that service node to publish the message to a `11/WAKU2-RELAY` pubsub topic.
Additionally, there is sometimes a need for confirmation that a message has been received "by the network"
(here, at least one node).

`WAKU2-LIGHTPUSH` is a request/response protocol for this.

## Payloads

```protobuf
syntax = "proto3";

message LightPushRequest {
        string request_id = 1;
    uin32 request_type = 10;  // reserved for future request type selection, currently it is always RELAY (0)
    string topic = 20;
    WakuMessage message = 21;
}

message LightPushResponse {
    string request_id = 1;
    uint32 status_code = 10; // non zero in case of failure, see appendix
    optional string status_desc = 11;
    uint32 relay_peer_count = 12;
}
```

### Message Relaying

Nodes that respond to `LightPushRequest` MUST either relay the encapsulated message via [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay) protocol on the specified `pubsub_topic` or `shard` ([WAKU2-RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md)) depending on the network configuration.
If they are unable to do so for some reason, they SHOULD return an error code in `LightPushResponse`.

### Examples of possible error codes

| Result | Code | Note |
|--------|------|------|
| SUCCESS  | 0 | Successfull push, response's relay_peer_count holds the number of peers the message is pushed.    |
| BAD_REQUEST | 400   |     |
| NO_PEERS_TO_RELAY | 404 | Service node has no relay peers  |
| PAYLOAD_TOO_LARGE | 413 | Message exceeds certain size limit, it can depend on network configuration, see status_desc for details.  |
| UNSUPPORTED_TOPIC | 415 | Requested push on pubsub_topic or shard is not possible as the service node is not subscibed to. |
| TOO_MANY_REQUESTS | 429 | DOS protection prevented this request as the current request exceeds the configured request rate. |
| INTERNAL_SERVER_ERROR  | 500 | status_desc holds explanation of the error.  |
| SERVICE_UNAVAILABLE | 503 | Lightpush service not available  |

> The list of error codes are not complete and can be extended in the future.

## Security Considerations

Since this can introduce an amplification factor, it is RECOMMENDED for the node relaying to the rest of the network to take extra precautions.
Therefore Waku2 applies or will apply:
- DOS protection through request rate limitation on the service itself.
- message rate limiting via [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay), applied via network membership subscription.

> These features are under development. 

## Future work

- Add support attaching RLN proof for the message requested to be relayed.
- Add support for other request types.
- Incentivization of the service

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

* [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay)
* [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay)
