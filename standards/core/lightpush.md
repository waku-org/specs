---
title: WAKU-LIGHTPUSH
name: Waku Light Push
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
A common use case is to request that the service node publish the message to an `11/WAKU2-RELAY` pubsub-topic.
Additionally, there is sometimes a need for confirmation that a message has been received "by the network"
(here, at least one node).

`WAKU-LIGHTPUSH` is a request/response protocol for this.

## Payloads

```protobuf
syntax = "proto3";

message LightPushRequest {
    string request_id = 1;
    // 10 Reserved for future `request_type`. Currently, RELAY is the only available service.
    optional string pubsub_topic = 20;
    WakuMessage message = 21;
}

message LightPushResponse {
    string request_id = 1;
    uint32 status_code = 10; // non zero in case of failure, see appendix
    optional string status_desc = 11;
    optional uint32 relay_peer_count = 12; // number of peers, message is successfully relayed to 
}
```

### Message Relaying

Nodes that respond to `LightPushRequest` SHOULD either relay the encapsulated message via [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay) protocol on the specified `pubsub_topic`
or perform another requested service.
Services beyond [11/WAKU2-RELAY](https://rfc.vac.dev/waku/standards/core/11/relay) are as yet underdefined.
Depending on the network configuration, the lightpush client may not need to provide `pubsub_topic` ([WAKU2-RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md)).
If the node is unable to do so for some reason, they SHOULD return an error code in `LightPushResponse`.
Once the relay is successful, the `relay_peer_count` will indicate the number of peers that the node has managed to relay the message to. It's important to note that this number may vary depending on the node subscriptions and support for the requested pubsub_topic. The client can use this information to either consider the relay as successful or take further action, such as switching to a lightpush service peer with better connectivity.
The field `relay_peer_count` may not be present or has the value zero in case of error or in other future use cases, where no relay is involved.

### Examples of possible error codes

| Result | Code | Note |
|--------|------|------|
| SUCCESS  | 200 | Successfull push, response's relay_peer_count holds the number of peers the message is pushed.    |
| BAD_REQUEST | 400   | Wrong request payload.    |
| PAYLOAD_TOO_LARGE | 413 | Message exceeds certain size limit, it can depend on network configuration, see status_desc for details.  |
| UNSUPPORTED_PUBSUB_TOPIC | 421 | Requested push on pubsub_topic is not possible as the service node does not support it. |
| TOO_MANY_REQUESTS | 429 | DOS protection prevented this request as the current request exceeds the configured request rate. |
| INTERNAL_SERVER_ERROR  | 500 | status_desc holds explanation of the error.  |
| NO_PEERS_TO_RELAY | 503 | Lightpush service is not available as the node has no relay peers. |

> The list of error codes is not complete and can be extended in the future.

## Security Considerations

Since this can introduce an amplification factor, it is RECOMMENDED for the node relaying to the rest of the network to take extra precautions.
Therefore Waku applies or will apply:
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
