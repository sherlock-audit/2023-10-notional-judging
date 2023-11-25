Gigantic Wintergreen Cheetah

medium

# reinvestReward() generates dust totalPoolClaim causing vault abnormal

## Summary

If the first user deposits too small , due to the round down, it may result in 0 shares, which will result in 0 shares no matter how much is deposited later. 
In `National`, this situation will be prevented by setting `a minimum borrow size and a minimum leverage ratio`. 
However, `reinvestReward()` does not have this restriction, which may cause this problem to still exist, causing the vault to enter an abnormal state.

## Vulnerability Detail

The calculation of the shares of the vault is as follows:
```solidity
    function _mintVaultShares(uint256 lpTokens) internal returns (uint256 vaultShares) {
        StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
        if (state.totalPoolClaim == 0) {
            // Vault Shares are in 8 decimal precision
@>          vaultShares = (lpTokens * uint256(Constants.INTERNAL_TOKEN_PRECISION)) / POOL_PRECISION();
        } else {
            vaultShares = (lpTokens * state.totalVaultSharesGlobal) / state.totalPoolClaim;
        }

        // Updates internal storage here
        state.totalPoolClaim += lpTokens;
        state.totalVaultSharesGlobal += vaultShares.toUint80();
        state.setStrategyVaultState();
```
If the first `deposit` is too small, due to the conversion to `INTERNAL_TOKEN_PRECISION`, the precision is lost, resulting in `vaultShares=0`. Subsequent depositors will enter the second calculation, but `totalVaultSharesGlobal=0`, so `vaultShares` will always be `0`.

To avoid this situation, `Notional` has restrictions.
>hey guys, just to clarify some rounding issues stuff on vault shares and the precision loss. Notional will enforce a minimum borrow size and a minimum leverage ratio on users which will essentially force their initial deposits to be in excess of any dust amount. so we should not really see any tiny deposits that result in rounding down to zero vault shares. If there was rounding down to zero, the account will likely fail their collateral check as the vault shares act as collateral and the would have none.
there is the possibility of a dust amount entering into depositFromNotional in a valid state, that would be due to an account "rolling" a position from one debt maturity to another. in this case, a small excess amount of deposit may come into the vault but the account would still be forced to be holding a sizeable position overall due to the minium borrow size. 



in `reinvestReward()`, not this limit
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
@>      state.totalPoolClaim += poolClaimAmount;
        state.setStrategyVaultState();

        emit RewardReinvested(rewardToken, amountSold, poolClaimAmount);
    }
```

From the above code, we know that `reinvestReward()` will increase `totalPoolClaim`, but will not increase `totalVaultSharesGlobal`.

This will cause problems in the following scenarios:
1. The current vault has deposits.
2. `Rewards` have been generated, but `reinvestReward()` has not been executed.
3. The `bot` submitted the `reinvestReward()` transaction. but step 4 execute first
4. The users  took away all the deposits `totalPoolClaim = 0`, `totalVaultSharesGlobal=0`.
5. At this time `reinvestReward()` is executed, then `totalPoolClaim > 0`, `totalVaultSharesGlobal=0`.
6. Other users' deposits will fail later

It is recommended that `reinvestReward()` add a judgment of `totalVaultSharesGlobal>0`.

Note: If there is a malicious REWARD_REINVESTMENT_ROLE, it can provoke this issue by donating reward token and triggering reinvestReward() before the first depositor appears.

## Impact

reinvestReward() generates dust totalPoolClaim causing vault abnormal 
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L385
## Tool used

Manual Review

## Recommendation

```diff
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
+       require(state.totalVaultSharesGlobal > 0 ,"invalid shares");
        state.totalPoolClaim += poolClaimAmount;
        state.setStrategyVaultState();

        emit RewardReinvested(rewardToken, amountSold, poolClaimAmount);
    }

```