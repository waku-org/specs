# Waku Specifications

Waku builds a family of privacy-preserving, censorship-resistant communication protocols for web3 applications.
This repository contains specifications for the Waku suite of protocols.

## List of Specifications:

- Standards
  - Core: Standards and protocols for the core Waku p2p communications offering.
  - Application: Standards and protocols that describe various applications or encryption use cases built on top of a Waku network.
- Informational: Waku design issues, general guidelines or background information that does not constitute a new feature.

### Core standards

| Waku Specifications | Description |
| ---- | -------------- |
|[10/WAKU2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)| Waku Overview |
|[11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)| Waku Relay |
|[12/WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md)| Waku Filter |
|[13/WAKU2-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md)| Waku Store |
|[14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md)| Waku Message |
|[17/WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)| Waku RLN Relay |
|[19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)| Waku Lightpush |
|[33/WAKU2-DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md)| Waku Discovery v5 Ambient Peer Discovery |
|[36/WAKU2-BINDINGS-API](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/36/bindings-api.md)| Waku C Bindings API |
|[WAKU2-PEER-EXCHANGE](standards/core/peer-exchange.md)| Waku Peer Exchange |
|[WAKU2-ENR](standards/core/enr.md)| Waku Usage of ENR |
|[WAKU2-INCENTIVIZATION](standards/core/incentivization.md)| Waku Incentivization |
|[WAKU2-METADATA](standards/core/metadata.md)| Waku Metadata |
|[64/WAKU2-NETWORK](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/64/network.md)| Waku Network |
|[RELAY-SHARDING](standards/core/relay-sharding.md)| Waku Relay Sharding |
| WAKU2-STOREV3 | Coming Soon |

### Application standards

| Waku Specifications | Description |
| ---- | -------------- |
|[20/TOY-ETH-PM](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/20/toy-eth-pm.md)| Toy Ethereum Private Message |
|[21/WAKU2-FAULT-TOLERANT-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/21/fault-tolerant-store.md)| Waku Fault-Tolerant Store |
|[22/TOY-CHAT](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/22/toy-chat.md)| Waku Toy Chat |
|[26/WAKU2-PAYLOAD](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/26/payload.md)| Waku Message Payload Encryption |
|[53/WAKU2-X3DH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/53/x3dh.md)| X3DH Usage for Waku Payload Encryption |
|[54/WAKU2-X3DH-SESSIONS](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/54/x3dh-sessions.md)| Session Management for Waku X3DH |
|[DANDELION](standards/application/dandelion.md)| Waku Dandelion |
|[WAKU2-DEVICE-PAIRING](standards/application/device-pairing.md)| Device Pairing and Secure Transfers with Noise |
|[WAKU2-NOISE](standards/application/noise.md)| Noise Protocols for Waku Payload Encryption |
|[TOR-PUSH](standards/application/tor-push.md)| Waku Tor Push |

### Informational

| Waku Specifications | Description |
| ---- | -------------- |
|[23/WAKU2-TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md)| Waku Topic Usage Recommendations |
|[27/WAKU2-PEERS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/27/peers.md)| Waku Peer Management Recommendations |
|[29/WAKU2-CONFIG](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/29/config.md)| Waku Parameter Configuration Recommendations |
|[30/ADAPTIVE-NODES](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/30/adaptive-nodes.md)| Adaptive Nodes |
|[ADVERSARIAL-MODELS](informational/adversarial-models.md)| Waku Adversarial Models and Attack-based Threat List |
|[RELAY-STATIC-SHARD-ALLOC](informational/relay-static-shard-alloc.md)| Waku Relay Static Shard Allocation |

## Resources
Relevant Waku resources related to the specifications located in this repository:
- [Waku.org](https://waku.org/)
- [nwaku: Waku Node](https://github.com/waku-org/nwaku)

## Contributions 

Contributions are welcome from any party. 
Contributors can create specifications relating to the Waku domain and
create a pull request to begin discussion.
The recommended [template](./template.md) may be used for new proposed specifications.

New specifications are considered a proof of concept.
Once a rough consensus is reached towards stabilization, 
the specification may be considered to receive the draft status and 
further discussion will continue on the [Vac RFC-Index](https://github.com/vacp2p/rfc-index) repository.

**NOTE:** Specifications located in this repository should be considered not production ready.
Discussion should be conducted with the intention of maturing each specification.

Head over to the [Vac RFC-Index](https://github.com/vacp2p/rfc-index) repository where other Waku specifications live.
