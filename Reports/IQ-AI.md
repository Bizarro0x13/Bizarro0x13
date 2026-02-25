# IQ-AI — Bug Reports
https://code4rena.com/audits/2025-01-iq-ai
---

# [M-1] Ineffective proposal threshold validation allows setting arbitrary high values

## Summary
The setProposalThresholdPercentage function in TokenGovernor checks the wrong variable when trying to enforce the maximum 10% threshold limit. Instead of validating the new incoming value, it checks the current stored value, making the protection completely ineffective.

This means anyone can propose and pass a governance proposal that sets the threshold to any value (even 100% or higher), which could effectively break the governance system by making it impossible for token holders to create new proposals.

The impact is severe because:

It could lock out all governance participation if set too high
It breaks the intended maximum 10% threshold protection
It could require another governance proposal to fix, which might be impossible if the threshold is set too high

## Proof of Concept
```js
function setProposalThresholdPercentage(uint32 _proposalThresholdPercentage) public {

    if (msg.sender != address(this)) revert NotGovernor();

    if (proposalThresholdPercentage > 1000) revert InvalidThreshold(); // Max 10%

    proposalThresholdPercentage = _proposalThresholdPercentage;

    emit ProposalThresholdSet(_proposalThresholdPercentage);

}
```
## Attack scenario:

Attacker creates a governance proposal to set proposalThresholdPercentage to 9000 (90%)
The check if (proposalThresholdPercentage > 1000) looks at the current value (let’s say 100), not 9000
The check passes because 100 < 1000
The threshold is set to 9000, requiring 90% of total supply to create proposals
The governance system becomes effectively frozen as gathering 90% support for new proposals is practically impossible

## Recommended Mitigation Steps
Change the validation to check the new input value instead of the current value:
```diff
function setProposalThresholdPercentage(uint32 _proposalThresholdPercentage) public {

    if (msg.sender != address(this)) revert NotGovernor();

-  if (proposalThresholdPercentage > 1000) revert InvalidThreshold(); // Max 10%

+   if (_proposalThresholdPercentage > 1000) revert InvalidThreshold(); // Max 10%

    proposalThresholdPercentage = _proposalThresholdPercentage;

    emit ProposalThresholdSet(_proposalThresholdPercentage);

}
```