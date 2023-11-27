Festive Misty Dog

high

# The protocol doesn't correctly account for AuraStash causing reward to be lost

## Summary
The protocol does not take into account the stash AURA rewards, which may result in this part of the rewards remaining in the protocol and unable to be withdrawn.

## Vulnerability Detail
Some Aura pools have two sources of AURA. First from the booster but also as a secondary reward. This secondary reward is stash AURA that doesn't behave like regular AURA. Although the protocol can obtain this part of the reward, it cannot be reinvested because the `_isInvalidRewardToken()` check will fail.
```solidity

  function reinvestReward(
        SingleSidedRewardTradeParams[] calldata trades,
        uint256 minPoolClaim
    ) external whenNotLocked onlyRole(REWARD_REINVESTMENT_ROLE) returns (
        address rewardToken,
        uint256 amountSold,
        uint256 poolClaimAmount
    ) {
        // Will revert if spot prices are not in line with the oracle values
        _checkPriceAndCalculateValue();

        // Require one trade per token, if we do not want to buy any tokens at a
        // given index then the amount should be set to zero. This applies to pool
        // tokens like in the ComposableStablePool.
        require(trades.length == NUM_TOKENS());
        uint256[] memory amounts;
        (rewardToken, amountSold, amounts) = _executeRewardTrades(trades);

        poolClaimAmount = _joinPoolAndStake(amounts, minPoolClaim);

        // Increase LP token amount without minting additional vault shares
        StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
        state.totalPoolClaim += poolClaimAmount;
        state.setStrategyVaultState();

        emit RewardReinvested(rewardToken, amountSold, poolClaimAmount);
    }


```

## Impact
stash AURA rewards will remain in the protocol.

## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L385-L411

## Tool used

Manual Review

## Recommendation

