Shallow Peanut Elk

high

# Reward tokens are re-entered during vault restoration

## Summary

Reward tokens are re-entered during vault restoration, resulting in unnecessary slippage being incurred during the re-enter AND the reward tokens are forced to re-enter the pool without consideration of the market condition, potentially leading to lower returns.

## Vulnerability Detail

Reward tokens (e.g. CRV, BAL, CVX, AURA) are sometimes also tokens of a pool. Following are some examples of such pools for Curve:

- crvUSD/ETH/CRV (https://curve.fi/#/ethereum/pools/factory-tricrypto-4/deposit)
- CRV/sdCRV (https://curve.fi/#/ethereum/pools/factory-v2-300/deposit)
- CRV/cvxCRV (https://curve.fi/#/ethereum/pools/factory-v2-283/deposit)
- ETH/CVX (https://curve.fi/#/ethereum/pools/cvxeth/deposit)

When an emergency exit, all LP tokens are redeemed for the underlying pool tokens. Subsequently, once the emergency is over, the `restoreVault` function will be executed to restore the vault. Further note from the source code's comment about the following intention/expectation of this function from the perspective of the protocol team, which are two (2) invariants that should be upheld:

1) Restores **withdrawn** tokens 
   - In other words, tokens that are not withdrawn during emergency exit must not be restored back. There is a good reason for this, as this control guards against potential donation + front-run attacks prior to the vault restoration.
2) Restore back into the vault **proportionally**
   - In other words, a single-sided or imbalance approach of adding liquidity back to the vault is not recommended as this approach often incur unnecessary slippage

Within the `restoreVault` function, based on the code at Line 512-514 and the source code's comment at Line 509, it was observed that all balances held by the vault will be used to re-enter.

However, the logic fails to consider a scenario where the reward token is the same as one of the pool tokens. Assume that before the emergency exit, there are 1000 CVX reward tokens. During the emergency exit, an additional 1000 CVX is redeemed.

When the `restoreVault` function is executed, all of the CVX tokens, including the 1000 CVX reward tokens, will be re-added back to the pool, which breaks the above two (2) invariants. 

Firstly, reward tokens that are not tokens withdrawn during an emergency exit are added back, breaking the first invariant. Secondly, the addition of CVX reward tokens will result in the deposit being imbalanced, breaking the second invariant where the deposit should be proportional. Suppose a proportional deposit is 1000 CVX + 1000 frxETH, with no or minimum slippage. However, due to this issue, the imbalance deposit of 2000 CVX + 1000 frxETH is made, suffering unnecessary slippage during the re-enter.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L514

```solidity
File: SingleSidedLPVaultBase.sol
498:     /// @notice Restores withdrawn tokens from emergencyExit back into the vault proportionally.
499:     /// Unlocks the vault after restoration so that normal functionality is restored.
500:     /// @param minPoolClaim slippage limit to prevent front running
501:     function restoreVault(
502:         uint256 minPoolClaim, bytes calldata /* data */
503:     ) external override whenLocked onlyNotionalOwner {
504:         StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
505: 
506:         (IERC20[] memory tokens, /* */) = TOKENS();
507:         uint256[] memory amounts = new uint256[](tokens.length);
508: 
509:         // All balances held by the vault are assumed to be used to re-enter
510:         // the pool. Since the vault has been locked no other users should have
511:         // been able to enter the pool.
512:         for (uint256 i; i < tokens.length; i++) {
513:             if (address(tokens[i]) == address(POOL_TOKEN())) continue;
514:             amounts[i] = TokenUtils.tokenBalance(address(tokens[i]));
515:         }
516: 
517:         // No trades are specified so this joins proportionally using the
518:         // amounts specified.
519:         uint256 poolTokens = _joinPoolAndStake(amounts, minPoolClaim);
520: 
521:         state.totalPoolClaim = state.totalPoolClaim + poolTokens;
522:         state.setStrategyVaultState();
523: 
524:         _unlockVault();
525:     }
```

## Impact

Following are some of the impact of this issue:

1) Suppose a proportional deposit is 1000 CVX + 1000 frxETH, with no or minimum slippage. However, due to this issue, the imbalance deposit of 2000 CVX + 1000 frxETH is made, suffering unnecessary slippage during the re-enter, leading to loss of assets.
2) The reward tokens are often slashed in the vault until a favorable market condition occurs or waiting for the right time/opportunity occurs. In this case, they will be re-entered into the pool to capture the maximum value for the vault shareholders. However, due to this bug, the reward tokens are forced to re-enter the pool without consideration of the market condition, potentially leading to lower returns.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L514

## Tool used

Manual Review

## Recommendation

Consider keeping track of the tokens received during an emergency exit. During vault restoration, only the amount recorded earlier should be re-entered. This also helps to guard against potential donation + front-run attacks before vault restoration, resulting in a large increase in the value of a vault share, which might cause unintended consequences.