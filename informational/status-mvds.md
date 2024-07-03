---
title: MVDS-STATUS
name: MVDS Usage in Status App
status: raw
category: Informational
tags: [waku/informational]
editor: Kaichao Sun <kaichao@status.im>
contributors:
---

## Abstract

This document lists the features that are using [MVDS](https://github.com/vacp2p/rfc-index/blob/main/vac/2/mvds.md) in the Status application.

## Background

Status app uses MVDS to ensure messages going through Waku are acknolwedged by the recipient. This is to ensure that the messages are not missed by any interested parties.


## Features using MVDS

Message types using MVDS:

- ApplicationMetadataMessage_EDIT_MESSAGE (ChatType == ChatTypeOneToOne || ChatType == ChatTypePrivateGroupChat)
- ApplicationMetadataMessage_DELETE_MESSAGE (ChatType == ChatTypeOneToOne || ChatType == ChatTypePrivateGroupChat)
- ApplicationMetadataMessage_PIN_MESSAGE (ChatType == ChatTypeOneToOne || ChatType == ChatTypePrivateGroupChat)
- ApplicationMetadataMessage_CHAT_MESSAGE (ChatType == ChatTypeOneToOne || ChatType == ChatTypePrivateGroupChat)

- ApplicationMetadataMessage_GROUP_CHAT_INVITATION

- ApplicationMetadataMessage_ACCEPT_CONTACT_REQUEST
- ApplicationMetadataMessage_CONTACT_UPDATE
- ApplicationMetadataMessage_RETRACT_CONTACT_REQUEST

- ApplicationMetadataMessage_DECLINE_CONTACT_VERIFICATION
- ApplicationMetadataMessage_ACCEPT_CONTACT_VERIFICATION
- ApplicationMetadataMessage_CANCEL_CONTACT_VERIFICATION
- ApplicationMetadataMessage_REQUEST_CONTACT_VERIFICATION

- ApplicationMetadataMessage_SEND_TRANSACTION (ChatType == ChatTypeOneToOne)
- ApplicationMetadataMessage_DECLINE_REQUEST_ADDRESS_FOR_TRANSACTION (ChatType == ChatTypeOneToOne)
- ApplicationMetadataMessage_DECLINE_REQUEST_TRANSACTION (ChatType == ChatTypeOneToOne)
- ApplicationMetadataMessage_ACCEPT_REQUEST_ADDRESS_FOR_TRANSACTION (ChatType == ChatTypeOneToOne)
- ApplicationMetadataMessage_REQUEST_ADDRESS_FOR_TRANSACTION (ChatType == ChatTypeOneToOne)
- ApplicationMetadataMessage_REQUEST_TRANSACTION (ChatType == ChatTypeOneToOne)

- ApplicationMetadataMessage_SYNC_CHAT_MESSAGES_READ
- ApplicationMetadataMessage_SYNC_VERIFICATION_REQUEST
- ApplicationMetadataMessage_SYNC_TRUSTED_USER
- ApplicationMetadataMessage_SYNC_ACCOUNT_CUSTOMIZATION_COLOR
- ApplicationMetadataMessage_SYNC_ENS_USERNAME_DETAIL
- ApplicationMetadataMessage_SYNC_BOOKMARK
- ApplicationMetadataMessage_SYNC_INSTALLATION_COMMUNITY
- ApplicationMetadataMessage_SYNC_INSTALLATION_CONTACT_V2
- ApplicationMetadataMessage_SYNC_CHAT_REMOVED
- ApplicationMetadataMessage_SYNC_CLEAR_HISTORY
- ApplicationMetadataMessage_SYNC_CHAT
- ApplicationMetadataMessage_SYNC_PAIR_INSTALLATION
- ApplicationMetadataMessage_SYNC_CONTACT_REQUEST_DECISION
- ApplicationMetadataMessage_SYNC_PROFILE_PICTURES
- ApplicationMetadataMessage_SYNC_KEYPAIR
- ApplicationMetadataMessage_SYNC_ACCOUNT
- ApplicationMetadataMessage_SYNC_ACCOUNTS_POSITIONS
- ApplicationMetadataMessage_SYNC_COLLECTIBLE_PREFERENCES
- ApplicationMetadataMessage_SYNC_TOKEN_PREFERENCES
- ApplicationMetadataMessage_SYNC_SAVED_ADDRESS
- ApplicationMetadataMessage_SYNC_PROFILE_SHOWCASE_PREFERENCES
- ApplicationMetadataMessage_SYNC_DELETE_FOR_ME_MESSAGE

- ApplicationMetadataMessage_SYNC_COMMUNITY_SETTINGS
- ApplicationMetadataMessage_COMMUNITY_SHARD_KEY

- ApplicationMetadataMessage_SYNC_ACTIVITY_CENTER_READ
- ApplicationMetadataMessage_SYNC_ACTIVITY_CENTER_UNREAD
- ApplicationMetadataMessage_SYNC_ACTIVITY_CENTER_ACCEPTED
- ApplicationMetadataMessage_SYNC_ACTIVITY_CENTER_COMMUNITY_REQUEST_DECISION
- ApplicationMetadataMessage_SYNC_ACTIVITY_CENTER_DELETED
- ApplicationMetadataMessage_SYNC_ACTIVITY_CENTER_DISMISSED

- ApplicationMetadataMessage_SYNC_SETTING



## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [MVDS](https://github.com/vacp2p/rfc-index/blob/main/vac/2/mvds.md)
