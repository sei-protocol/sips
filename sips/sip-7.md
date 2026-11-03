**SIP-7: Decrease the Unbonding Time From 21 to 7 Days**

| SIP Number | 7 |
| ----- | ----- |
| Title | Decrease the Unbonding Time From 21 to 7 Days |
| Description | Decrease the risk for SEI stakers: change the pacific-1 unbonding time from 21 days to 7 days with one governance proposal. |
| Author | [Alexander H.](mailto:alexanderh@seinetwork.io) |
| Type | Standard (Core) |
| Created | 10/26/2026 |
| Status | Draft |
| Comments | TBD |

## **Abstract**

This SIP reduces the unbonding time on pacific-1 from 21 days to 7 days. This lowers the liquidity risk of staking for SEI holders, because a staker who exits gets their tokens back 14 days sooner.

The change takes one governance proposal that updates the `staking/UnbondingTime` parameter. No binary upgrade is needed. Undelegations and redelegations that are already in progress when the proposal executes keep their original 21-day completion time.

At 7 days, unbonding is still longer than the 3-day governance voting period and the 48-hour evidence max age. Node operators who use state sync must set their trust period to 112 hours or less, which is two-thirds of the new unbonding time.

## **Motivation**

Staked SEI gets rewards, and it protects the chain against attacks. To exit, a staker must unbond the tokens. During the unbonding time, the tokens get no rewards, and the staker cannot sell or move them.

On pacific-1, the unbonding time is 21 days. In the past, investors saw this as a risk, because of the volatility of crypto prices. For these holders, the long unbonding time is a reason to stake less or not at all.

With 7 days, staking carries less investor risk against price volatility. Liquid staking also becomes faster and uses less capital. [Security Considerations](#security-considerations) explains why the chain continues to stay secure with 7 days unbonding.

## **Specification**

### **Parameter Change**

| Parameter | Current value | New value |
| --- | --- | --- |
| `staking/UnbondingTime` | `1814400s` (21 days) | `604800s` (7 days) |

A single `ParameterChangeProposal` sets the new value. The proposal encodes the duration in nanoseconds, so the value is `"604800000000000"`. The chain only checks that the value is positive. It does not compare it with any other parameter.

### **Transition**

In this SIP, H is the block in which the proposal executes, at the end of its voting period, and X is the block time of H:

- Undelegations and redelegations submitted in block H or earlier use 21 days, because their transactions execute before the proposal does.
- Undelegations and redelegations submitted after block H use 7 days.
- A validator that leaves the active set in block H or later uses 7 days.
- A redelegation away from a validator that is still unbonding inherits that validator's completion time, which may be a 21-day one. A redelegation away from an unbonded validator completes immediately, as it does today.

An entry's completion time is fixed when the entry is created and is never recalculated. Entries with both unbonding times therefore exist side by side until X + 21 days. The chain releases entries in order of completion time, so a 7-day entry created after H can complete before a 21-day entry created earlier. Integrators should read completion times from the chain rather than compute them.

### **Default Trust Period for State Sync**

A state sync light client does not trust a header older than its trust period. The trust period must be shorter than the unbonding time, with enough time left to detect and report misbehavior while the validators that signed the header are still bonded or unbonding. The current default is 168 hours, which is exactly 7 days and would leave no time at all.

sei-chain v6.8 fixes the trust period at 112 hours and removes the setting, so operators can no longer configure it. 112 hours is two-thirds of the new unbonding time, which leaves 56 hours to report misbehavior. A node running v6.8 or later needs no change and ignores any `trust-period` value still in its config file. A node that still runs v6.7 or earlier must set `[statesync] trust-period` to 112 hours or less before X (refer to [Validators and Node Operators](#validators-and-node-operators)).

### **Out of Scope**

This SIP does not change the evidence parameters, the voting periods or the slashing parameters. It also does not change the completion time of any entry that is open at block H.

## **Rationale**

### **Why 7 Days**

The unbonding time has to stay longer than each of these time limits on pacific-1:

| Time limit | Value | Why unbonding must be longer |
| --- | --- | --- |
| Expedited voting period | 1 day | Governance can pass a response to an attack before the chain releases the stake behind it |
| Evidence window | 48 hours | Stake stays slashable for any evidence the chain accepts |
| Voting period | 3 days | Governance can pass a response to an attack before the chain releases the stake behind it |
| State sync trust period (fixed in v6.8) | 112 hours (about 4.7 days) | Signers of a trusted header cannot leave before it expires |

Seven days clears all three with room to spare: 4 days over the voting period and 5 days over the evidence window. It also leaves enough time to notice an attack and respond, even over a weekend or a holiday.

Going below 7 days would shrink the margin over the voting period, and would need a sei-chain release to lower the state sync trust period first. Anything between 7 and 21 days keeps more of the risk this SIP sets out to reduce, without buying meaningful extra safety against the limits above.

### **Alternatives Considered**

- **Keep 21 days.** Rejected: stakers keep carrying three weeks of price risk on exit, which is the problem this SIP addresses.
- **Keep 21 days and rely on liquid staking for fast exits.** Rejected: a liquid staking token gives an instant exit only at its market price, which can trade below redemption value. Redemptions still go through the unbonding queue, so liquid staking stays slow as well (refer to [Liquid Staking](#liquid-staking)).
- **A shorter value, such as 3 to 5 days.** Rejected: the margins over the 3-day voting period and the 112-hour trust period shrink to hours, or disappear.
- **An intermediate value, such as 14 days.** Rejected: it keeps most of the exit risk and buys no meaningful extra safety, because 7 days already clears every limit in [Why 7 Days](#why-7-days).
- **Change the value in a software upgrade.** Rejected: it is a single parameter, so a governance proposal is enough. A proposal also lets validators vote on the exact value, with no binary upgrade.

## **Economic Considerations**

This section describes the economic effects qualitatively. It does not rely on a formal economic model.

### **Risk for Stakers**

While tokens are unbonding, the staker carries the full SEI price risk and cannot act on it. At 7 days that window is 14 days shorter. The size of the risk shrinks too. If daily price moves are independent, the expected size of the price change grows with the square root of time, so over 7 days it is approximately 58% of what it is over 21 days.

A shorter unbonding time matters most to funds and other holders who need to be able to sell at short notice, because it lets them stake a larger share of their SEI. Every staker can also react faster to events, such as a problem with their validator.

### **Liquid Staking**

Liquid staking protocols pay out redemptions through the unbonding queue, so a 7-day unbonding time makes them faster and more capital efficient:

- A redemption takes 7 days instead of 21 days.
- A delegator can have at most 7 open unbonding entries per validator. At 21 days, a protocol can open about one entry every 3 days per validator, so redemptions are batched and users wait longer. At 7 days, it can open one entry a day.
- Protocols that offer instant redemption from an unstaked SEI reserve can keep that reserve smaller and stake more.
- A trader who buys a liquid staking token below its redemption value and redeems it carries the price risk until the redemption completes. A shorter unbonding time makes that trade cheaper and less risky, which keeps the token's market price closer to its redemption value.
- Lending protocols that accept liquid staking tokens as collateral can turn seized collateral into SEI in 7 days instead of 21.

### **Redelegation**

After a redelegation, the delegator cannot redelegate the same stake away from the new validator until the redelegation completes. With a 7-day unbonding time parameter, delegators can move the same stake between validators three times as often, so they can respond faster to a validator's performance or commission.

## **Security Considerations**

### **Slashing and Evidence**

The chain punishes a validator that double signs while its stake is bonded or unbonding. Stake that starts to unbond after the double sign stays slashable until its unbonding completes. The chain accepts evidence for 48 hours after the double sign; this is the evidence window. (Evidence also has a 100,000-block limit. It expires only once it is past both limits, and 100,000 blocks is about 13 hours, so 48 hours is the effective window.) Because 7 days is longer than 48 hours, all stake behind any evidence the chain accepts can still be punished.

On pacific-1, `slash_fraction_double_sign` and `slash_fraction_downtime` are both 0. Today, a double sign jails and tombstones the validator but burns no stake. The unbonding time therefore matters for slashing only if governance sets a non-zero fraction later. This SIP does not change the slashing parameters.

### **Governance**

Governance counts only bonded stake, measured at the end of the voting period. Stake that is unbonding does not count.

The voting period is 3 days, or 1 day for an expedited proposal. Both are shorter than 7 days. Tokens that start to unbond during a vote stay locked until after the vote ends, so they cannot be moved to another account and used to vote again on the same proposal (vote hopping).

A voter's stake counts only if it stays bonded until the vote ends. An undelegation then takes another 7 days, so a voter cannot sell until at least 7 days after the vote ends. The 7 days run from the end of the vote. The result can come later, because the chain can spread the tally of a large vote over several blocks.

### **Economic Security**

An attacker needs more than one-third of the voting power to halt the chain and more than two-thirds to control it. The unbonding time does not change these fractions or the amount of SEI they represent. If this SIP leads to more SEI being staked, buying enough stake for an attack becomes more expensive.

What does change is how long an attacker's own stake stays locked after an attack: at least 7 days instead of 21. With the slash fraction at 0, this lock is the main economic cost for an attacker who owns their stake, because they cannot sell into the price drop that an attack would likely cause. A shorter lock reduces that cost. A non-zero slash fraction would still need evidence within the 48-hour evidence window, as it does today.

An attacker can also control voting power without owning most of the stake behind it, by compromising validator keys or by running a malicious validator. The lock, and any slash, then falls mostly on the delegators. The attacker's own exposure is at most the validator's self-bond, and nothing at all if the keys were stolen. With a short position, the attacker could even profit from the price drop. The unbonding lock barely touches this attacker, so shortening it changes little here. Against this attacker, the chain relies on operators keeping their keys secure, and on double-sign evidence, which jails and tombstones the validator. Delegators can also move their stake to another validator. A redelegation takes effect at once and does not wait for the unbonding time.

A former validator's keys have no stake behind them once its unbonding completes: after 7 days instead of 21. Because the trust period stays shorter than the unbonding time, a syncing node only relies on signatures from validators whose stake is still bonded or unbonding (refer to [Light Clients and State Sync](#light-clients-and-state-sync)).

### **Light Clients and State Sync**

A light client must not trust a header older than its trust period. At 7 days, the old default of 168 hours would leave no time to report misbehavior. sei-chain v6.8 fixes the trust period at 112 hours. Nodes on v6.7 or earlier must set `[statesync] trust-period` to 112 hours or less (refer to [Default Trust Period for State Sync](#default-trust-period-for-state-sync)).

A node that has been offline for more than 7 days must not trust unknown peers when it syncs again. It must sync from trusted peers or from a recent trusted header. Today, this limit is 21 days.

IBC is retired on pacific-1. No other chain runs a light client of Sei, so no external trust periods are affected.

## **Impact and Migration**

### **Delegators and Liquid Staking Users**

No action is needed. Undelegations that start after block H complete after 7 days, and entries that are already open at block H keep their completion time.

Because of this, an undelegation started in the 14 days before X can complete later than one started right after block H.

### **Protocols and Contracts**

A contract that hardcodes 21 days usually keeps working, but the chain returns principal to it after 7 days. A contract that books balance increases as rewards can count that principal as rewards.

Do not lower a hardcoded value before X + 21 days, because until then the chain still has open 21-day entries. A contract that switches to 7 days too early expects tokens that the chain has not released yet. What happens next depends on the contract's design: its withdrawals may revert, or it may book the shortfall as a slash.

To change a hardcoded value:

1. Keep the value at 21 days until X + 21 days.
2. Check that the contract has no entry still open from block H or earlier.
3. Change the value to 7 days, or better, read it from the chain.

An EVM contract can read the current value, in seconds, from [`params().unbondingTime`](https://github.com/sei-protocol/sei-chain/blob/118658041c754a04e36b21abf3df2f865cf6051e/precompiles/staking/Staking.sol#L430-L431) on the staking precompile. For the completion time of an undelegation, as a Unix timestamp in seconds, call [`unbondingDelegation()`](https://github.com/sei-protocol/sei-chain/blob/118658041c754a04e36b21abf3df2f865cf6051e/precompiles/staking/Staking.sol#L253).

One liquid staking protocol on pacific-1 hardcodes 21 days, and its team has agreed to keep that value until X + 21 days. If your contract depends on the unbonding time, please comment on this SIP.

### **Wallets, Explorers, Exchanges and Custodians**

Do not display a fixed 21 days. Read the current value from `/cosmos/staking/v1beta1/params` or from the staking precompile. Until X + 21 days, show each entry's own completion time instead of computing one.

### **Validators and Node Operators**

1. If your node uses state sync, run v6.8 or later. If it still runs v6.7 or earlier, set `[statesync] trust-period` to 112 hours or less before X (refer to [Default Trust Period for State Sync](#default-trust-period-for-state-sync)).
2. When you state sync, use a trust height less than 112 hours old.
3. Before you vote, check that the proposal sets the value to `604800000000000`.

This SIP does not change how much disk space nodes use.

## **References**

- [SIP-1](./sip-1.md): Sei Improvement Proposal Process
- [Staking precompile interface](https://github.com/sei-protocol/sei-chain/blob/118658041c754a04e36b21abf3df2f865cf6051e/precompiles/staking/Staking.sol) in sei-chain

## **Copyright**

CC0 1.0.
