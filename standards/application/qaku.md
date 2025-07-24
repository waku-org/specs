---
title: WAKU-APP-QAKU
name: Waku App Qaku
category: Standards Track
tags: waku
editor: 
contributors: 
- Jimmy Debe <jimmy@status.im>
---

## Abstract

The document describes the Qaku web application bulit with Waku.
The use of Waku LightPush ....

## Background

Waku is a family of robust,
censorship-resistant communication protocols designed to enable privacy-focused messaging for web3 apps.

Qaku Q&A (Questions & Answers over Waku)
is an application allowing you to create a Q&A board,
share a link and let users post questions.
Users can upvote questions which move to the top of the list on the board.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”,
“SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and
“OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

The Q&A board is created and maintained by an owner,
who SHOULD support an accessible interface for its users.

### Identity

- Users and an owner MUST generate a private key
- The owner MUST generate a private key to before becoming the owner of the Q&A board.
- The owner address, derived from the public key, SHOULD be made public as the owner of Qaku.
- A `paasword` SHOULD be used to encrypt the private key

The public key is used to sign and verify messages.
The owner MUST be able to enable and disable questions added to the board.

### Node

- Qaku uses
[19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)
- A new `content_topic` is created when a new Q&A board is created.
- All questions and answer are pushed to the content_topic

### Messages

Messages sent on the `content_topic` are identified by their message type.
Message types in Qaku include:

- Question Message
- Answered Message
- Control Message
- Upvote Message
- Moderation Message
- Poll messag

#### Question Message

```js
{
  "QuestionMessage" : [
    "question": string,
    "timestamp": number,
    "author?": identity,
    "delegationInfo?": delegationInfo
  ]
}

```

#### Answered Message

```js
{
  "AnswerMessage" : [
    hash: string,
    text: string,
    timestamp: number,
    delegationInfo?: delegationInfo
  ]  
}

```

#### Control Message

```js
{
  "ControlMessage" : [
    title: string,
    id: string,
    enabled: boolean,
    timestamp: number,
    owner: string,
    admins: string[],
    moderation: boolean,
    description: string,
    updated: number,
    startDate: number,
    endDate?: number,
    allowsParticipantsReplies: number,
    delegationInfo?: DelegationInfo
  ]
}

```
#### Upvote Message

```js
{
  "UpVoteMessage" : [
    questionId: string,
    hash: string,
    type: UpvoteType,
    timestamp: number,
    delegationInfo?: DelegationInfo;
  ]
}
```

When a new question is created, the interface SHOULD lis


## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References
