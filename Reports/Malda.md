# Malda — Bug Reports
https://audits.sherlock.xyz/contests/1029
---

## [M-1] Incorrect Max Transfer Size Check in sendMsg Function

### Description
In the sendMsg function, the contract enforces a maximum transfer size per time window for each (destination chain, token) pair. However, there is a logic flaw in how the max transfer size is checked:

TransferInfo memory transferInfo = currentTransferSize[_msg.dstChainId][_msg.token];
uint256 transferSizeDeadline = transferInfo.timestamp + transferTimeWindow;
if (transferSizeDeadline < block.timestamp) {
    currentTransferSize[_msg.dstChainId][_msg.token] = TransferInfo(_amount, block.timestamp);
} else {
    currentTransferSize[_msg.dstChainId][_msg.token].size += _amount;
}

uint256 _maxTransferSize = maxTransferSizes[_msg.dstChainId][_msg.token];
if (_maxTransferSize > 0) {
    require(transferInfo.size + _amount < _maxTransferSize, Rebalancer_TransferSizeExcedeed());
}
When the time window has expired (transferSizeDeadline < block.timestamp), currentTransferSize is reset to the new _amount.
However, the subsequent max transfer size check still uses the old transferInfo.size (from before the reset), not the updated value.

### Impact
Rebalancer may be unable to perform valid rebalancing transfers after a time window reset, leading to unnecessary failed transactions and potential disruption of protocol operations.

### POC
If currentTransferSize.size was 100e18, _amount is 50e18, and maxTransferSize is 125e18, and the time window has expired, the new transfer should be allowed (since 50e18 < 125e18).
But the check uses 100e18 + 50e18 = 150e18, which incorrectly exceeds the limit and causes a revert.

### Recommendation
Update the max transfer size check to use the correct, updated value after resetting currentTransferSize.