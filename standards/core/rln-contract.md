---
title: WAKU2-RLN-CONTRACT
name: Waku2 RLN Contract Specification
category: Standards Track
tags: [waku/core-protocol]
editor: Sergei Tikhomirov <sergei@status.im>
contributors:
---

## Abstract

This specification describes the smart contract that governs RLN memberships, in particular:

- the overall smart contract functionality;
- contract parameters;
- the values of those parameters for the initial deployment on a mainnet chain;
- the way in which the parameters can be modified.

## Background

[Rate-Limiting Nullifiers](https://rate-limiting-nullifier.github.io/rln-docs/) (RLN) is a ZK-based rate-limiting technique used in Waku.
Before sending messages, users must register in a smart contract.
They then attach a ZK proof of valid membership to every message.
Relaying nodes check message validity and drop invalid messages.

A message is considered valid in terms of RLN if:
- its sender has an active RLN membership; and
- the sender hasn't exceeded their rate limit within the current epoch.

RLN is managed by a smart contract that keeps track of active memberships.
Users interact with the contract to register a new membership and
to obtain Merkle proofs and the tree root,
which is necessary for proof generation and verification.

In testnet deployment, the RLN contract does not require any payment apart from gas costs.
This document aims to make the RLN contract suitable for mainnet deployment.
In particular, the contract would require users to put down a deposit.
The aim of the deposit initially is purely to protect the system from denial-of-service attacks using bandwidth capping.
It may also be explored as a revenue stream for Waku in later versions.

## Semantics

This section is not intended to describe the full contract functionality.
We only focus on functions required for managing memberships.
The document might later evolve into a full-fledged contract specification.

The RLN contract provides the following functionalities:
- register a membership;
- extend a membership;
- terminate the membership (withdraw the deposit).

For the _Contract Owner_, the contract provides the following additional functionality:
- change the modifiable parameters (see parameter table below);
- (TBD) freeze certain functionality (e.g., in case of an attack, stop new registrations, deposit withdrawals, etc).

The deposit is not taken away (slashed) for exceeding the rate limit.

### Membership lifecycle

A membership can be in one of the following states:
- active;
- grace-period;
- expired;
- erased.

A user locks up a deposit to register a membership, which starts its lifecycle in the _active_ state.
After the expiration term (see parameter table), a membership moves to the _grace period_ state.
During grace period, the membership owner can either extend the membership or withdraw the deposit.
Deposit withdrawal makes a _grace-period_ membership _expired_ immediately.
Extension makes a _grace-period_ membership _active_ immediately.
A _grace-period_ membership becomes _expired_ automatically after its grace period ends.
The slot in the RLN tree of an _expired_ membership can be overridden by a new membership.
An _expired_ membership becomes _erased_ when its tree slot is overridden.

The user can only extend their membership during its grace period.
Otherwise, the user assumes the risk of the membership being erased at any moment.

The user cannot withdraw their deposit from an _active_ membership.
The rationale for this limitation is to prevent users from abusing the contract by making deposits and withdrawals in short succession.

Memberships are not transferable.
One Ethereum address MAY register multiple memberships.
One Waku node MAY manage multiple memberships,
although this functionality is not yet implemented.

Availability of user functionalities depending on the membership state is as follows:

|                       | Active | Grace Period | Expired | Erased |
| --------------------- | ------ | ------------ | ------- | ------ |
| Send a message        | Yes    | Yes          | Yes     | No     |
| Extend the membership | No     | Yes          | No      | No     |
| Withdraw the deposit  | No     | Yes          | Yes     | Yes    |

Note that the user can still send messages when their membership is _expired_.
This is explained by the fact that Relay nodes cannot verify the state of the membership, only its presence in the RLN tree.
_Expired_ memberships are not erased from the tree proactively, as this would require someone to send a transaction and pay the gas costs.
Instead, an _expired_ memberships is only erased when a new memberships overtakes its slot.
We expect that an honest user would not want to risk their membership being erased at any time, and would either extend or terminate it during its grace period.

We do not allow extending an _active_ membership.
The rationale here is that if the _Owner_ changes some contract parameters (e.g., for security purposes),
users with extended memberships will not be affected by the changes for a long time.

### User actions

In more detail, user actions are handled as follows.

#### Register a membership

Registering a membership:
1. Check whether there are _expired_ memberships.
	1. If there are _expired_ memberships:
		1. Select the _expired_ membership with the earliest end of its grace period;
		2. Move that membership to the _erased_ state;
		3. Create a new _active_ membership using the erased membership's tree slot.
	2. If there are no _expired_ memberships, check whether the total limits are respected.
		1. If the new membership would exceed the maximum total number of _active_, _grace-period_, and _expired_ memberships, fail the registration;
		2. If the new membership would exceed the maximum total rate limit of _active_, _grace-period_, and _expired_ memberships, fail the registration;
		3. Otherwise, create a new membership in the _active_ state.

#### Send a message

Note: when sending a message, the user only interacts with a Relay node, not with the smart contract.
The Relay node, in turn, interacts with the smart contract to check the user-provided RLN proof.

Sending a message:
1. If the message is committed to a different epoch than the current epoch, drop the message;
2. If the user has exceed their allowed rate limit for the current epoch, drop the message;
3. If the RLN proof fails, drop the message;
4. Otherwise, propagate the message.

#### Extend a membership

(TBD) The contract doesn't check whether the transaction sender is the owner of the membership in question.
In other words, any user is permitted to extend any _grace-period_ membership.

Note: an attacker can extend other users' _grace-period_ memberships without their consent,
which has at least two negative consequences:
1. users who planned to withdraw their deposit later would have to wait for another membership term;
2. if an attacker extends many memberships, the cap on total active rate limit can be exceeded, preventing the registration of new memberships.

Extending a membership:
1. Check whether the membership is in its _grace-period_ state:
	1. If it is not, fail the transaction;
	2. Otherwise, mark the membership as _active_.

#### Withdraw the deposit

Withdrawing the deposit:
1. (TBD) Check whether the user is the "owner" of the membership:
	1. If not, fail the transaction;
	2. Otherwise, continue.
2. Check whether the membership is in one of the states that allow withdrawal:
	1. If not, fail the transaction;
	2. Otherwise, continue.
3. Check whether the deposit has not yet been withdrawn:
	1. If not, fail the transaction;
	2. Otherwise, withdraw the deposit.

(TBD) If an _expired_ membership becomes _erased_,
and its deposit has not been withdrawn,
data is saved in the smart contract that would allow the user to withdraw the deposit later.

## Contract governance and mutability

The contract governance is as follows:

1. the first deployment of the contract allows the _Owner_ to modify certain parameters (TBD) under certain constraints (TBD);
2. at some point, the _Owner_ SHOULD renounce their privileges, and the contract MUST become immutable;
3. further upgrades, if necessary, can be done by deploying a new contract and migrating the membership set.

## Parameters

We make the following design choices:
- there are `3` membership tiers: low-, mid-, and high-tier;
- prices are quoted in `USD`;
- payment is accepted in `DAI`;
- the price for a given tier is linearly proportional to its rate limit.

For the initial deployment, we suggest the following values.

| Parameter                                                 | Symbol     | Value   | Units                                                                                | Modifiable? | Comment                        |
| --------------------------------------------------------- | ---------- | ------- | ------------------------------------------------------------------------------------ | ----------- | ------------------------------ |
| Epoch length                                              | `epoch`    | `10`    | minutes                                                                              | Yes         |                                |
| Maximum total rate limit of concurrent active memberships | `R_{max}`  | TBD     | messages per `epoch`                                                                 | Yes         | Can be replaced with `N_{max}` |
| Maximum number of concurrent active memberships           | `N_{max}`  | `10000` | units                                                                                | Yes         | Can be replaced with `R_{max}` |
| Rate limit for low-tier                                   | `R_{low}`  | `20`    | messages per `epoch`                                                                 | TBD         |                                |
| Rate limit for mid-tier                                   | `R_{mid}`  | `200`   | messages per `epoch`                                                                 | TBD         |                                |
| Rate limit for high-tier                                  | `R_{high}` | `600`   | messages per `epoch`                                                                 | TBD         |                                |
| Unit price                                                | `p_u`      | `0.01`  | `USD` per `1` message per `epoch` for the duration of one membership expiration term | Yes         |                                |
| Membership expiration term                                | `T`        | `90`    | days                                                                                 | Yes         |                                |
| Grace period                                              | `G`        | `30`    | days                                                                                 | Yes         |                                |

### Discussion on some parameter values

#### Epoch length

Epoch length is a global parameter set in the smart contract.
Rate limits are defined in terms of the maximum allowed messages per epoch.
There is a trade-off between short and long epochs.

On the one hand, longer epochs allow for short-term spikes,
which allows for more flexibility.
Arguably, short-term bursts followed by a period of silence is a common pattern for some use cases.
Usage peaks are expected to average out over longer time periods,
allowing us to reason about network utilization on a longer time scale.

On the other hand, long epochs lead to more memory requirements for nodes due to the growth of the nullifier log.
Each message contains a nullifier that proves its validity in terms of RLN.
Within the latest epoch, each relaying node stores all nullifiers to detect users exceeding the rate limit.
Nullifier log data needs to be stored in memory.
Each nullifier plus metadata is `128` bytes (per message).
This means that one user with a `1` message per second rate limit (high-tier) generates up to `600 * 128 / 1024 = 75 KiB` of nullifier log data per 10-min epoch.
This corresponds to:
- for 1000 users: approximately `73 MiB`;
- for 10 thousand users: approximately `732 MiB`.

#### Maximum total message limit / concurrently active memberships

We may want to limit the total rate limit available to all users.
The rationale is that the total network bandwidth is a limited resource.
The total active rate limit should not exceed the network's real capabilities.

There may be two ways to implement a global cap:
- limit the total rate limits;
- limit the total number of active memberships (e.g., at 10K memberships).

#### Proportional pricing

We suggest proportional pricing for simplicity.
Proportional pricing means that if tier A offers N times the limit of tier B, then tier A is N times more expensive than tier B.

An alternative approach could be bulk discounts for high-tier memberships.
Discounts are more efficient but promote centralization.
Finding the right trade-off remains subject for future work.

#### Accepted tokens

When choosing a token to accept, we consider the following criteria:

- a stablecoin: USD-denominated pricing is simpler for users and for developers (avoiding an oracle);
- popular, high liquidity;
- preferably decentralized;
- with a reasonably good track record w.r.t. censorship.

Based on these criteria, we suggest accepting DAI for the initial deployment.
Other tokens may be added in the future.

#### Grace period and membership extension

Membership extension does not require a new deposit.
However, the opportunity cost of locked-up capital plus gas fees for extension transactions make extensions non-free.
Moreover, a legitimate user is unlikely to risk their membership being taken over at any moment by not renewing their membership.
We argue that no new deposit requirement for membership extension is sufficient for the initial mainnet deployment.

## Implementation Suggestions

The current version of the contract (RLNv2) is deployed on Sepolia testnet ([source code](https://github.com/waku-org/waku-rlnv2-contract/blob/main/src/WakuRlnV2.sol)).

## Security / Privacy Considerations

Requesting a Merkle proof for one's own membership through a third-party RPC provider may endanger the requester's privacy.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [17/WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)

