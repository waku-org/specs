---
title: TEMPLATE
name: Specification Template
category: (Standards Track|Informational|Best Current Practice)
tags: an optional list of tags, not standard
editor: Jimmy Debe <jimmy@status.im>
contributors:
---

### (Info, remove this section)

This section contains meta info about writing specifications.
This section (including its subsections) MUST be removed.

[COSS](https://rfc.vac.dev/spec/1/) explains the Vac RFC process.

### Tags

The `tags` metadata SHOULD contain a list of tags if applicable.

* `core` for Waku protocol definitions
* `application` for applications built on top of Waku protocol,
* `informational` for general guidelines, background information etc. 

## Abstract


## Background / Rationale / Motivation

This section serves as an introduction providing background information and a motivation/rationale for the specification.

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
