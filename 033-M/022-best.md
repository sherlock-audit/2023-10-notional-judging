Festive Misty Dog

medium

# Bypassing access controls for `REWARD_REINVESTMENT_ROLE` in `claimRewardTokens()`

## Summary
The `claimRewardTokens()` function, is designed to be accessible only to entities with the `REWARD_REINVESTMENT_ROLE`.However ,anyone can claim Convex rewards for any account, bypassing the intended role-based access control.

## Vulnerability Detail
The `SingleSidedLPVaultBase.claimRewardTokens()` function is intended to allow the claiming of rewards, restricted to entities with the `REWARD_REINVESTMENT_ROLE`. However, there is a potential vulnerability as anyone can claim [Convex](https://etherscan.io/address/0x0A760466E1B4621579a82a39CB56Dda2F4E70f03#code) rewards for any account, bypassing the intended role-based access control.
```solidity
function getReward(address _account, bool _claimExtras) public updateReward(_account) returns(bool){
    uint256 reward = earned(_account);
    if (reward > 0) {
        rewards[_account] = 0;
        rewardToken.safeTransfer(_account, reward);
        IDeposit(operator).rewardClaimed(pid, _account, reward);
        emit RewardPaid(_account, reward);
    }

    //also get rewards from linked rewards
    if(_claimExtras){
        for(uint i=0; i < extraRewards.length; i++){
            IRewards(extraRewards[i]).getReward(_account);
        }
    }
    return true;
}

```

## Impact
Bypass the intended role-based access control.

## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L376-L378

## Tool used

Manual Review

## Recommendation

