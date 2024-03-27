# Waku Specifications

Waku builds a family of privacy-preserving, censorship-resistant communication protocols for web3 applications.
This repository contains specifications for the Waku suite of protocols.

## List of Specifications:
- Informational: Waku design issues, general guidelines or background information that does not constitute a new feature.
- Standards
  - Core: Standards and protocols for the core Waku p2p communications offering.
  - Application: Standards and protocols that describe various applications or encryption use cases built on top of a Waku network.

| Waku Specifications | Description |
| ---- | -------------- |
|[10/WAKU2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)| Core |
|[11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)| Core |
|[12/WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md)| Core|
|[13/WAKU2-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md)| Core |
|[14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md)| Core |
|[15/WAKU2-BRIDGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/15/bridge.md)| Core |
|[16/WAKU2-RPC](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/16/rpc.md)| Core |
|[17/WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)| Core |
|[18/WAKU2-SWAP](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/18/swap.md)| Application |
|[19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)| Core |
|[20/TOY-ETH-PM](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/20/toy-eth-pm.md)| Application |
|[21/WAKU2-FAULT-TOLERANT-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/21/fault-tolerant-store.md)| Application |
|[22/TOY-CHAT](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/22/toy-chat.md)| Informational |
|[23/WAKU2-TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md)| Informational |
|[26/WAKU2-PAYLOAD](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/26/payload.md)| Application |
|[27/WAKU2-PEERS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/27/peers.md)| Informational |
|[29/WAKU2-CONFIG](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/29/config.md)| Informational |
|[30/ADAPTIVE-NODES](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/30/adaptive-nodes.md)| Informational |
|[33/WAKU2-DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md)| Core |
|[36/WAKU2-BINDINGS-API](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/36/bindings-api.md)| Core |
|[53/WAKU2-X3DH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/53/x3dh.md)| Application |
|[54/WAKU2-X3DH-SESSIONS](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/54/x3dh-sessions.md)| Application |
|[ADVERSARIAL-MODELS](informational/adversarial-models.md)| Informational |
|[RELAY-STATIC-SHARD-ALLOC](informational/relay-static-shard-alloc.md)| Informational |
|[DANDELION](standards/application/dandelion.md)| Application |
|[WAKU2-DEVICE-PAIRING](standards/application/device-pairing.md)| Application |
|[WAKU2-PEER-EXCHANGE](standards/core/peer-exchange.md)| Core |
|[WAKU2-ENR](standards/core/enr.md)| Core |
|[WAKU2-NOISE](standards/application/noise.md)| Application |
|[TOR-PUSH](standards/application/tor-push.md)| Application |
|[WAKU2-INCENTIVIZATION](standards/core/incentivization.md)| Core |
|[WAKU2-METADATA](standards/core/metadata.md)| Core |
|[WAKU2-NETWORK](standards/core/network.md)| Core |
|[RELAY-SHARDING](standards/core/relay-sharding.md)| Core |
| WAKU2-STOREV3 | Coming Soon |



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
