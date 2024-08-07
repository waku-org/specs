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
We only focus on functions required for managing memberships, expiration, deposits, etc.
The document might later evolve into a full-fledged contract specification.

The RLN contract provides the following functionalities:
- register a membership in a given tier;
- extend a membership;
- withdraw the deposit for an expired membership.

For the _Contract Owner_, the contract provides the following additional functionality:
- change the modifiable parameters (see parameter table below);
- (TBD) freeze certain functionality (e.g., in case of an attack, stop new registrations, deposit withdrawals, etc).

In the initial deployment:
- there is no slashing - that is, a user's deposit is not taken away in case of misbehavior;
- membership expiration includes a grace period;
- a user can extend their membership without putting up any additional deposit.

### Membership life-cycle

To register a membership, a user locks up a deposit.
Each membership has an expiration date.
After a membership expires, it enters a grace period, during which the user can extend it.
Extending a membership requires no additional deposit.
After the grace period, an expired membership can be taken over by another user.
At any point, a user can withdraw their deposit and terminate the membership.

In more detail, the membership life-cycle is as follows:
1. A user registers a membership:
    1. If there are expired unclaimed slots in the tree, use the slot with the earliest expiration date past its grace period, otherwise create a new slot;
    2. if the membership re-uses a previously used slot, the previous user can claim their deposit back.
2. The user sends messages while the membership is active:
    1. If the user exceeds the rate limit, their over-limit messages are not propagated; the deposit is not slashed.
3. After the membership expires, the user can do one of the following:
    1. Do nothing. The deposit remains in the contract, and the user retains the ability to send messages (respecting the per-epoch rate limit) until another user takes their slot;
    2. Withdraw the deposit. The user receives the full deposit back, and loses the ability to send messages;
    3. Extend the membership. The user sends a transaction to re-enable their membership for another expiration term. The user only covers gas costs and does not have to provide additional funds. The original deposit remains in the contract. The membership is prolonged for another expiration term under the same conditions.

The user can extend the membership both during and after its grace period.
By not extending the membership before the grace period expires, the user assumes the risk of the membership being taken over at any moment.

A user can hold multiple memberships.
Memberships are not transferable.
There is no limitation on the number of membership per user (apart from the global limits, see parameter table).

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
| Unit price                                                | `p_u`      | TBD     | `USD` per `1` message per `epoch` for the duration of one membership expiration term | Yes         |                                |
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

On the one hand, memberships must expire,
otherwise a one-time payment would give a user the right to potentially infinite resource usage.
On the other hand, the “soft” expiration suggested above is likely sufficient
while balancing abuse prevention and ease of implementation:

- although extending a membership doesn’t require a new deposit, keeping the original deposit locked up plus paying gas fees imposes cost on membership extension;
- a legitimate user is unlikely to risk their membership being taken over at any moment by not renewing their membership.

## Implementation Suggestions

The current version of the contract (RLNv2) is deployed on Sepolia testnet ([source code](https://github.com/waku-org/waku-rlnv2-contract/blob/main/src/WakuRlnV2.sol)).

## Security / Privacy Considerations

Requesting a Merkle proof for one's own membership through a third-party RPC provider may endanger the requester's privacy.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)
- [17/WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)

