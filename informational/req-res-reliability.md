---
title: REQ-RES-RELIABILITY
name: Request-response protocols reliability
category: Best Current Practice
tags: [informational]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
---

## Abstract
This RFC describes set of instructions used across different [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md?plain=1#L3) implementations for improved reliability in request-response protocols such as [WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/19/lightpush.md?plain=1#L3C11-L3C26), [WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md?plain=1#L3) and [WAKU2-STORE](https://github.com/waku-org/specs/blob/a637d4bb34243dbd8f6771e0dee65669764f6798/standards/core/store.md?plain=1#L2).

## Motivation

Descriptions of mentioned protocols do not define some of the real world use cases that are oftenly observed in unreliable network environment. Such use cases can be: recovery from offline state, decrease rate of missed messages, increase probability of messages being broadcasted.

## Theory / Semantics

This section SHOULD explain in detail how the proposed protocol works.
It may touch on the wire format where necessary for the explanation.
This section MAY also specify endpoint behaviour when receiving specific messages, e.g. the behaviour of certain caches etc.

## Wire Format Specification / Syntax
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, 
“NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

This section SHOULD not contain explanations of semantics and focus on concisely defining the wire format.
Implementations SHOULD adhere to these exact formats to interoperate with other implementations.
It is fine, if parts of the previous section that touch on the wire format are repeated.
The purpose of this section is having a concise definition of what an implementation sends and accepts.
Parts that are not specified here are considered implementation details. 
Implementors are free to decide on how to implement these details.


## Implementation Suggestions (optional)
An optional *implementation suggestions* section may provide suggestions on how to approach implementation details, and, 
if available, point to existing implementations for reference.


## (Further Optional Sections)


## Security/Privacy Considerations

If there are none, this section MAY state that fact.
This section MAY contain additional relevant information, e.g. an explanation as to why there are no security consideration for the respective document.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

A list of references.
