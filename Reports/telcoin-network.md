# telcoin — Bug Reports
https://cantina.xyz/code/26d5255b-6f68-46cf-be55-81dd565d9d16/overview
---

## [H-1] Incorrect Assignment in Delegation Struct in delegateStake

### Finding Description
In the delegateStake function, the contract records a new delegation by assigning values to the delegations mapping:

```js
delegations[validatorAddress] =
    Delegation(blsPubkeyHash, msg.sender, validatorAddress, validatorVersion, nonce + 1);
```
Here, msg.sender is set as the validator and validatorAddress is set as the delegator in the Delegation struct. This is contrary to the intended logic, where validatorAddress should be the validator and msg.sender should be the delegator.

Because of this wrong assigning of delegator address, the real delegator would not be able to claimStakeRewards or unstake.
```js
function claimStakeRewards(address validatorAddress) external override whenNotPaused nonReentrant {
        // require validator is whitelisted, having been issued a ConsensusNFT by governance
        _checkConsensusNFTOwner(validatorAddress);
        uint8 validatorVersion = validators[validatorAddress].stakeVersion;

        // require caller is either the validator or its delegator
1 >      address recipient = _getRecipient(validatorAddress);
2 >      if (msg.sender != validatorAddress && msg.sender != recipient) revert NotRecipient(recipient);
        uint256 rewards = _claimStakeRewards(validatorAddress, recipient, validatorVersion);

        emit RewardsClaimed(recipient, rewards);
    }
At 1 we can see that the recipient is fetched from _getRecipient Function

function _getRecipient(address validatorAddress) internal view returns (address) {
        Delegation storage delegation = delegations[validatorAddress];
@>      address recipient = delegation.delegator;
        if (recipient == address(0x0)) recipient = validatorAddress;

        return recipient;
    }
```
The recipient is set as the delegator, which will be the validatorAddress.

In the 2nd location the check confirms that the msg.sender is either validatorAddress or the delegator(which will be the validatorAddress). So the function call will revert with NotRecipient error.

### Impact Explanation
This can result in delegator not being able to claimStakeRewards or unstake.

### Likelihood Explanation
Likelihood of this happening is High because everytime a user calls the delegateStake function, the delegator address is stored incorrectly.

### Proof of Concept
Suppose Alice (0xAlice) wants to delegate to Bob (0xBob):

Expected: delegations[0xBob] = Delegation(..., delegator=0xAlice, validator=0xBob, ...)
Actual: delegations[0xBob] = Delegation(..., delegator=0xBob, validator=0xAlice, ...)
This means Bob is incorrectly set as the delegator and Alice as the validator.

### Recommendation
delegations[validatorAddress] =

## [M-1] BLS Public Key Reuse Due to Missing Registration in Constructor

### Finding Description
In the ConsensusRegistry contract, the constructor initializes the initial set of validators and their associated data. However, it does not mark the BLS public keys of these initial validators as "used" in the usedBLSPubkeys mapping. This mapping is intended to prevent the reuse of BLS public keys, which is enforced in the _recordStaked function for all subsequent validators

if (usedBLSPubkeys[blsPubkeyHash]) revert DuplicateBLSPubkey();
usedBLSPubkeys[blsPubkeyHash] = true;
However, in the constructor, this line is missing when setting up the initial validators.

### Impact Explanation
Impact is High, since the BLS public keys of the initial validators are not marked as used, an attacker could later reuse one of these keys to register a new validator.

### Likelihood Explanation
Likelihood is High.

### Proof of Concept
Suppose the initial validator set includes a validator with BLS public key X. Since usedBLSPubkeys[X] is not set to true in the constructor, a malicious actor could later call stake(X) and successfully register a new validator with the same BLS key, which should not be possible.

### Recommendation
In the constructor, after processing each initial validator, add the following line to mark their BLS public key as used:

usedBLSPubkeys[blsPubkeyHash] = true;

## [M-2] The closeCase function will revert when there is no cachedUnsettledFrozen amount.

### Summary
The RecoverableWrapper::closeCase function will revert when the user does not have cachedUnsettledFrozen amount.

### Finding Description
User wraps 100e18, 150e18, 200e18 and 250e18 at different times.

so the queue would look like this: [{100e18, 1000, 0, 0, 2}, {150e18, 1500, 0, 1, 3}, {200e18, 2000, 0, 2, 4}, {250e18, 2500, 0, 3, 0}]

And the _unsettledRecords[User1] = {queue, head = 1, tail = 4, cacheIndex = 0}

_accountState[User1] = {700e18, 4, 0, 0}

Owner freezes 50e18 of user1 at 1st and 2nd Index. As the rawIndex(1) is greater than cacheIndex(0) so there would be no change in the cachedUnsettledFrozen.

queue: [{100e18, 1000, 50e18, 0, 2}, {150e18, 1500, 50e18, 1, 3}, {200e18, 2000, 0, 2, 4}, {250e18, 2500, 0, 3, 0}]

Now At 1501 User tries to unwrap 150e18 tokens.

After the unwrap the State would be

queue = [{100e18, 1000, 50e18, 0, 2}, {150e18, 1500, 50e18, 0, 3}, {200e18, 2000, 0, 0, 4}, {250e18, 2500, 0, 3, 0}]

_unsettledRecords[User1] = {queue, head =3, tail = 4, cacheIndex = 4}

_accountState[User1] = {550e18, 4, 450e18, 0}, Here:

cachedUnsettled = 450e18 because the 3rd and 4th indexes hold 200e18 and 250e18, respectively
cachedUnsettledFrozen = 0 because those records are not frozen
Owner then calls the closeCase(Suspension[{user1, 1 ,50e18},{user1, 2, 50e18}] function to unfreeze the index1 and index2 of the user.

This attempts to unfreeze index 1 and 2 for the user.

However, during execution:

rawIndex = 1 (and 2)
cacheIndex = 4
Since rawIndex < cacheIndex, the function attempts to reduce cachedUnsettledFrozen by amount

if (rawIndex <= _unsettledRecords[account].cacheIndex) {
    _accountState[account].cachedUnsettledFrozen -= amount;
}
But cachedUnsettledFrozen is 0, so this underflow will revert the transaction.
```js
function closeCase(
        bool recover,
        address victim,
        Suspension[] memory freezes
    )
        external
        virtual
        onlyOwner
        returns (bool)
    {
        for (uint256 i = 0; i < freezes.length; i++) {
            uint256 rawIndex = freezes[i].rawIndex;
            address account = freezes[i].account;
            uint128 amount = freezes[i].amount;
            Record memory r = _unsettledRecords[account].getAt(rawIndex);

            if (r.settlementTime == 0) {
                revert RecordNotFound(account, rawIndex);
            }
            uint256 head = _unsettledRecords[account].head;

            // checks that there is enough frozen to unfreeze. only deletes if past settlement time and is before head.
            // should also delete if queue is empty
            bool del = rawIndex < head || _unsettledRecords[account].isEmpty();
            _unsettledRecords[account].unfreezeRecord(rawIndex, amount, del);

            frozen[account] -= amount;
            // remove from cachedUnsettledFrozen
            if (rawIndex <= _unsettledRecords[account].cacheIndex) {
                _accountState[account].cachedUnsettledFrozen -= amount;
            }

            if (recover) {
                // spend the now-unfrozen record
                if (!del) {
                    _unsettledRecords[account].decrementRecordAmount(rawIndex, amount);
@>                  if (rawIndex <= _unsettledRecords[account].cacheIndex) {
                        _accountState[account].cachedUnsettled -= amount;
                    }
                }
                uint256 toRawIndex = _unsettledRecords[victim].enqueue(amount, block.timestamp + recoverableWindow);

                _accountState[account].balance -= amount;
                _accountState[victim].balance += amount;
                _accountState[account].nonce++;
                _accountState[victim].nonce++;
                bool pastSettlement = r.settlementTime <= block.timestamp;
                emit Transfer(account, victim, pastSettlement ? 0 : amount, pastSettlement ? amount : 0, toRawIndex);
            }
        }
        emit CaseClosed(recover, victim, freezes);
        return true;
    }
```
### Impact Explanation
Impact is high because the owner would not be able to unfreeze the frozen amount ever.

### Likelihood Explanation
Likelihood of this happening is high because it is not necessarily common that at at both freeze and closeCase time the rawIndex will remain less than cacheIndex.

### Recommendation
Do not include decrease of the cachedUnsettledFrozen in the closeCase function