# Security Audit Report

**Prepared by:** 0xapple  
**Date:** January 2026  
**Classification:** Confidential



## About the Auditor

0xapple is an independent smart contract security researcher specializing in the EVM ecosystem. He has over one year of professional experience conducting private smart contract security assessments for decentralized finance (DeFi) protocols. His work focuses on comprehensive manual code reviews, protocol-level threat modeling, and the identification of economic attack vectors prior to mainnet deployment.

Across multiple engagements, 0xapple has reviewed more than 21,000 normalized source lines of Solidity (nSLOC), identifying **2 Critical**, over **11 High**, and more than **22 Medium** severity vulnerabilities.

In addition to private audits, 0xapple is an active researcher on Sherlock, where he has achieved multiple Top-10 contest rankings and reported high-impact vulnerabilities across a variety of DeFi architectures, including vault systems, staking protocols, stablecoin mechanisms, cross-chain infrastructure, and governance frameworks.

## About the Protocol

This protocol is an EVM-compatible staking and governance system that enables token holders to stake assets in return for yield rewards and participation in on-chain decision-making. The audited codebase comprises six core contracts, totaling approximately **1,317** normalized source lines of code (nSLOC) in Solidity.

| Contract            | nSLOC | Description                                      |
|---------------------|-------|--------------------------------------------------|
| StakingPool.sol     | 423   | Deposit, withdraw, reward checkpointing, fee accrual |
| Governance.sol      | 389   | Proposal creation, voting, quorum, execution     |
| RewardDistributor.sol | 312 | Validator registry, reward notifications, distribution |
| GovernanceToken.sol | 98    | Token with mint/burn and delegation support      |
| StakingMath.sol     | 61    | Shared reward and fee calculation library        |
| Errors.sol          | 34    | Custom error and event definitions               |
| **Total**           | **1,317** |                                              |



## Executive Summary

0xapple conducted a security assessment of the Staking & Governance Protocol smart contracts from **January 20 to January 28, 2026**. The review focused on the core logic for token staking, reward distribution, validator management, and on-chain governance.

The audit examined approximately **1,317** normalized source lines of code (nSLOC) in Solidity, spanning six contracts. The assessment was performed manually through line-by-line review, combined with static analysis and targeted scenario testing, following industry-standard security auditing practices.

The review identified a total of **14 findings** categorized by severity:

- **3 High** severity issues: material risks that could lead to loss of funds, incorrect reward distribution, improper governance outcomes, or other direct economic impact
- **4 Medium** severity issues: logic errors, edge-case mishandlings, or suboptimal implementations
- **7 Low** severity issues: best-practice recommendations, minor optimizations, missing validations, gas suggestions, etc.

All findings include detailed descriptions and recommended remediations. The protocol team is encouraged to address the **high-severity** items as a priority before mainnet deployment, followed by a fix-review verification.

This report is provided under strict NDA and reflects the state of the codebase at the time of review. No public repository or commit identifier is referenced.

| Severity | Count |
|----------|-------|
| **High**   | 3     |
| **Medium** | 4     |
| **Low**    | 7     |



## Findings Summary

| ID    | Title                                                                 | Contract          | Severity |
|-------|-----------------------------------------------------------------------|-------------------|----------|
| H-01  | Staker Can Deflate Governance Quorum by Unstaking After Voting       | Governance        | **High**   |
| H-02  | Governance Voting Power Read from Live Balance Enables Flash-Stake Takeover | Governance        | **High**   |
| H-03  | Fee Share Calculation Overmints Shares to Fee Recipient on Every Accrual | StakingMath       | **High**   |
| M-01  | Removed Validators Can Still Claim Accrued Staking Rewards           | RewardDistributor | **Medium** |
| M-02  | Reward Rate Miscalculated When New Rewards Added Before Distribution Ends | RewardDistributor | **Medium** |
| M-03  | Protocol Loses Accumulated Fees During Epoch Reward Distribution     | StakingPool       | **Medium** |
| M-04  | Pending Rewards Permanently Lost When User Increases Stake           | StakingPool       | **Medium** |
| L-01  | Missing Zero Address Check in Validator Registration                 | RewardDistributor | **Low**    |
| L-02  | Governance Proposal Execution Has No Timelock                        | Governance        | **Low**    |
| L-03  | No Cap on Validator Count Allows Unbounded Registry Growth           | RewardDistributor | **Low**    |
| L-04  | claimRewards() Does Not Revert on Zero Reward Balance                | RewardDistributor | **Low**    |
| L-05  | Staking Contract Does Not Validate Minimum Deposit Amount            | StakingPool       | **Low**    |
| L-06  | settleEpoch() Can Be Called Multiple Times in the Same Epoch         | StakingPool       | **Low**    |
| L-07  | removeValidator() Does Not Emit an Event                             | RewardDistributor | **Low**    |


<br>

# Findings



## H-01: Staker Can Deflate Governance Quorum by Unstaking After Voting

**Severity:** High   
**Contract:** `Governance.sol`

### Description

When a staker casts a vote through `castVote()`, there is no mechanism that locks their staked balance for the duration of the voting period.

A user can vote with their full balance and immediately call `withdraw()` to unstake. This reduces the **total staked supply**, while their vote remains permanently counted.

Since `getQuorum()` reads the **current total staked supply at execution time**, coordinated unstaking after voting can artificially reduce the quorum requirement. This allows proposals to pass with significantly less economic backing than intended.

Votes are never invalidated after unstaking and the proposal outcome cannot be reversed.

```solidity
function castVote(uint256 proposalId, bool support) external {
    uint256 weight = getVotingPower(msg.sender);
    votes[proposalId][msg.sender] = weight;
    // no lock placed on msg.sender staked balance
}
```

### Recommendation

Introduce a vote-locking mechanism that prevents stakers from calling `withdraw()` while they have active votes on open proposals.

Track each user's locked balance per proposal and release the lock only after the proposal reaches a terminal state.

### Resolution
Fixed



## H-02: Governance Voting Power Read from Live Balance Enables Flash-Stake Takeover

**Severity:** High   
**Contract:** `Governance.sol`

### Description

The governance module calculates voting power using the **live staking balance at vote time** rather than a historical snapshot.

`getVotingPower()` reads `stakingContract.balanceOf(voter)` when the vote is cast. This allows tokens staked **after the proposal was created** to fully participate in voting.

An attacker can therefore:

1. Borrow tokens via flash loan
2. Stake them immediately before voting
3. Cast a vote with the full borrowed balance
4. Unstake and repay the loan in the same transaction or shortly after

This results in **zero economic exposure while exercising full governance control**, enabling attacks such as:

* treasury drainage
* malicious contract upgrades
* permanent governance parameter changes

```solidity
function getVotingPower(address voter) public view returns (uint256) {
    return stakingContract.balanceOf(voter); // live balance, no snapshot
}
```

### Recommendation

Implement **snapshot-based voting power**.

At proposal creation:

* Record a snapshot block number
* Use historical balances when computing voting power

Example:

```
stakingContract.getPastBalance(voter, snapshotBlock)
```

Additionally enforce a **minimum staking lockup period** before tokens become eligible for governance participation.

### Resolution
Fixed


## H-03: Reward Per Share Calculated Against Post-Deposit Supply Dilutes Existing Staker Rewards

**Severity:** High   
**Contract:** `StakingMath.sol`

### Description

In `notifyRewardAmount()`, the reward-per-share accumulator is calculated using the **total supply after new deposits have already been recorded**, instead of the supply at the time rewards were earned.

As a result, newly deposited tokens participate in reward distribution that they were **never entitled to**, diluting rewards for existing stakers.

Example scenario:

* Existing supply: 1000
* New rewards: 100
* New deposit: 200

Incorrect calculation:

```
rewardPerShare = (100 * PRECISION) / (1000 + 200)
               = 0.0833 per token
```

Existing stakers collectively receive **83.33 rewards instead of 100**.

Correct calculation:

```
rewardPerShare = (100 * PRECISION) / 1000
               = 0.1 per token
```

### Recommendation

Snapshot the total supply **before processing new deposits** and use that value when updating the reward-per-share accumulator.

### Resolution
Fixed


## M-01: Removed Validators Can Still Claim Accrued Staking Rewards

**Severity:** Medium   
**Contract:** `RewardDistributor.sol`

### Description

`removeValidator()` deletes the validator from the active registry but leaves the `validatorRewards` mapping untouched.

This allows removed validators to still call `claimRewards()` and withdraw their entire accrued reward balance.

A validator removed for malicious behavior such as downtime or double-signing can therefore bypass any intended penalty.

```solidity
function removeValidator(address validator) external onlyGovernance {
    delete activeValidators[validator];
    // validatorRewards[validator] remains intact
}
```

### Recommendation

On validator removal:

* Update the `validatorRewards` mapping
* Redirect accrued rewards to a designated address such as the staking pool.

### Resolution
Fixed

## M-02: Reward Rate Miscalculated When New Rewards Added Before Distribution Ends

**Severity:** Medium   
**Contract:** `RewardDistributor.sol`

### Description

When `notifyRewardAmount()` is called while a previous reward period is still active, the remaining rewards are recalculated using `block.timestamp`.

Small delays between the intended call time and the actual execution time can therefore reduce the effective reward rate.

The discrepancy accumulates silently across many distribution cycles with no reconciliation mechanism.

```solidity
uint256 remaining = periodFinish - block.timestamp;
uint256 leftover = remaining * rewardRate;

rewardRate = (amount + leftover) / rewardDuration;
```

### Recommendation

Store the exact remaining reward amount when the distribution period is initialized instead of recomputing it from elapsed time.

Emit an event when the recomputed value deviates from the stored value beyond a threshold.

### Resolution
Fixed

## M-03: Protocol Loses Accumulated Fees During Epoch Reward Distribution

**Severity:** Medium   
**Contract:** `StakingPool.sol`

### Description

`getFeeAmount()` calculates protocol fees using the **current reserve balance**.

During epoch settlement, `settleEpoch()` transfers a large portion of the reserve balance before fees are claimed.

As a result, fees that accrued against the **pre-settlement balance** are permanently lost.

```solidity
function settleEpoch() external onlyKeeper {
    uint256 rewardAmount = (reserveBalance() * epochDistributionRate) / 100;
    _transferReserveToDistributor(rewardAmount); // balance reduced
}
```

### Recommendation

Accrue fees before transferring rewards:

```
_accrueAndStoreFees();
```

Alternatively, refactor fee calculation to use a **time-weighted balance accumulator** instead of instantaneous balance.

### Resolution
Fixed

## M-04: Pending Rewards Permanently Lost When User Increases Stake

**Severity:** Medium   
**Contract:** `StakingPool.sol`

### Description

When a user increases their stake using `deposit()`, `_updateRewardCheckpoint()` is called before pending rewards are calculated.

This overwrites the checkpoint that tracks the user's previous reward index, causing previously accrued rewards to be lost.

Users increasing their position therefore lose all rewards accumulated since their last interaction.

```solidity
function deposit(uint256 amount) external {
    _updateRewardCheckpoint(msg.sender);
    stakedBalances[msg.sender] += amount;
}
```

### Recommendation

Always compute pending rewards **before updating checkpoints**.

Suggested pattern:

1. Calculate pending rewards
2. Store them in `pendingRewards[msg.sender]`
3. Update the checkpoint
4. Update the staked balance

### Resolution
Fixed

## L-01: Missing Zero Address Check in Validator Registration

**Severity:** Low   
**Contract:** `RewardDistributor.sol`

### Description

`registerValidator()` does not validate that the provided address is non-zero.

A zero address entry would corrupt the validator registry and any rewards directed to it would be unrecoverable.

### Recommendation

Add a validation check:

```solidity
require(validator != address(0), "ZeroAddress");
```



## L-02: Governance Proposal Execution Has No Timelock

**Severity:** Low   
**Contract:** `Governance.sol`

### Description

Proposals can be executed immediately once voting ends.

Without a timelock, users have **no reaction window** to respond to potentially harmful governance decisions.

### Recommendation

Introduce a mandatory execution delay of **24–48 hours** between proposal passage and execution.



## L-03: No Cap on Validator Count Allows Unbounded Registry Growth

**Severity:** Low   
**Contract:** `RewardDistributor.sol`

### Description

`registerValidator()` imposes no upper bound on the number of validators.

Over time this can cause loops in reward distribution to become excessively expensive and potentially exceed block gas limits.

### Recommendation

Introduce a constant such as:

```
MAX_VALIDATORS
```

and revert when the registry limit would be exceeded.



## L-04: `claimRewards()` Does Not Revert on Zero Reward Balance

**Severity:** Low   
**Contract:** `RewardDistributor.sol`

### Description

Calling `claimRewards()` with zero rewards silently succeeds without reverting or emitting an event.

This wastes gas and complicates monitoring of reward claims.

### Recommendation

Add a check:

```solidity
require(rewardAmount > 0, "NoRewardsToClaim");
```



## L-05: Staking Contract Does Not Validate Minimum Deposit Amount

**Severity:** Low   
**Contract:** `StakingPool.sol`

### Description

`deposit()` accepts any non-zero amount including extremely small deposits (dust).

These dust positions increase storage usage and complicate accounting without providing meaningful rewards.

### Recommendation

Introduce a minimum deposit threshold and revert when the amount is below it.



## L-06: `settleEpoch()` Can Be Called Multiple Times in the Same Epoch

**Severity:** Low   
**Contract:** `StakingPool.sol`

### Description

`settleEpoch()` does not verify whether the current epoch has ended before executing distribution logic.

Repeated calls within the same epoch could trigger multiple partial distributions and corrupt accounting.

### Recommendation

Add a validation check:

```solidity
require(
    block.timestamp >= lastEpochSettlement + EPOCH_DURATION,
    "EpochNotFinished"
);
```


## L-07: `removeValidator()` Does Not Emit an Event

**Severity:** Low   
**Contract:** `RewardDistributor.sol`

### Description

`removeValidator()` updates state without emitting an event, unlike `registerValidator()`.

This prevents off-chain systems from reliably tracking validator removal history.

### Recommendation

Add and emit an event:

```solidity
event ValidatorRemoved(address indexed validator, uint256 timestamp);
```


# Disclaimer
This report is for informational purposes only and does not constitute legal, financial, or investment advice. Findings reflect the state of the code at the time of review and are based solely on manual analysis of the in-scope contracts. No guarantee is made that all vulnerabilities have been identified. The auditor assumes no liability for any losses, and the protocol team is responsible for implementing and verifying all remediations prior to deployment.