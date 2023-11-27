Shallow Peanut Elk

high

# Fewer than expected LP tokens if the pool is imbalanced during vault restoration

## Summary

The vault restoration function intends to perform a proportional deposit. If the pool is imbalanced due to unexpected circumstances, performing a proportional deposit is not optimal. This results in fewer pool tokens in return due to sub-optimal trade, eventually leading to a loss for the vault shareholder.

## Vulnerability Detail

Per the comment on Line 498, it was understood that the `restoreVault` function intends to deposit the withdrawn tokens back into the pool proportionally.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L501

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
..SNIP..
```

The main reason to join with all the pool's tokens in exact proportions is to minimize the price impact or slippage of the join. If the deposited tokens are imbalanced, they are often swapped internally within the pool, incurring slippage or fees.

However, the concept of proportional join to minimize slippage does not always hold with the current implementation of the `restoreVault` function.

**Proof-of-Concept**

1) At T0, assume that a pool is perfectly balanced (50%-50%) with 1000 WETH and 1000 stETH.
2) At T1, an emergency exit is performed, the LP tokens are redeemed for the underlying pool tokens proportionally, and 100 WETH and 100 stETH are redeemed
3) At T2, certain events happen or due to ongoing issues with the pool (e.g., attacks, bugs, mass withdrawal), the pool becomes imbalanced (30%-70%) with 540 WETH and 1260 stETH.
4) At T3, the vault re-enters the withdrawn tokens to the pool proportionally with 100 WETH and 100 stETH. Since the pool is already imbalanced, attempting to enter the pool proportionally (50% WETH and 50% stETH) will incur additional slippage and penalties, resulting in fewer LP tokens returned.

This issue affects both Curve and Balancer pools since joining an imbalanced pool will always incur a loss.

**Explantation of imbalance pool**

A Curve pool is considered imbalanced when there is an imbalance between the assets within it. For instance, the Curve stETH/ETH pool is considered imbalanced if it has the following reserves:

- ETH: 340,472.34 (31.70%)
- stETH: 733,655.65 (68.30%)

If a Curve Pool is imbalanced, attempting to perform a proportional join will not give an optimal return (e.g. result in fewer Pool LP tokens received).

In Curve Pool, there are penalties/bonuses when depositing to a pool. The pools are always trying to balance themselves. If a deposit helps the pool to reach that desired balance, a deposit bonus will be given (receive extra tokens). On the other hand, if a deposit deviates from the pool from the desired balance, a deposit penalty will be applied (receive fewer tokens).

The following is the source code of `add_liquidity` function taken from https://github.com/curvefi/curve-contract/blob/master/contracts/pools/steth/StableSwapSTETH.vy. As shown below, the function attempts to calculate the `difference` between the `ideal_balance` and `new_balances`, and uses the `difference` as a factor of the fee computation, which is tied to the bonus and penalty.

```python
def add_liquidity(amounts: uint256[N_COINS], min_mint_amount: uint256) -> uint256:
..SNIP..
    if token_supply > 0:
        # Only account for fees if we are not the first to deposit
        fee: uint256 = self.fee * N_COINS / (4 * (N_COINS - 1))
        admin_fee: uint256 = self.admin_fee
        for i in range(N_COINS):
            ideal_balance: uint256 = D1 * old_balances[i] / D0
            difference: uint256 = 0
            if ideal_balance > new_balances[i]:
                difference = ideal_balance - new_balances[i]
            else:
                difference = new_balances[i] - ideal_balance
            fees[i] = fee * difference / FEE_DENOMINATOR
            if admin_fee != 0:
                self.admin_balances[i] += fees[i] * admin_fee / FEE_DENOMINATOR
            new_balances[i] -= fees[i]
        D2 = self.get_D(new_balances, amp)
        mint_amount = token_supply * (D2 - D0) / D0
    else:
        mint_amount = D1  # Take the dust if there was any
..SNIP..
```

Following is the mathematical explanation of the penalties/bonuses extracted from Curve's Discord channel:

- There is a “natural” amount of D increase that corresponds to a given total deposit amount; when the pool is perfectly balanced, this D increase is optimally achieved by a balanced deposit. Any other deposit proportions for the same total amount will give you less D.
- However, when the pool is imbalanced, a balanced deposit is no longer optimal for the D increase.

## Impact

There is no guarantee that a pool will always be balanced. Historically, there have been multiple instances where the largest curve pool (stETH/ETH) has become imbalanced (Reference [#1](https://twitter.com/LidoFinance/status/1437124281150935044) and [#2](https://www.coindesk.com/markets/2022/06/17/biggest-steth-pool-almost-empty-complicating-exit-for-would-be-sellers/)).

If the pool is imbalanced due to unexpected circumstances, performing a proportional deposit is not optimal, leading to the deposit resulting in fewer LP tokens than possible due to the deposit penalty or slippage due to internal swap.

The side-effect is that the vault restoration will result in fewer pool tokens in return due to sub-optimal trade, eventually leading to a loss of assets for the vault shareholder.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L501

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L48

## Tool used

Manual Review

## Recommendation

Consider providing the callers the option to deposit the reward tokens in a "non-proportional" manner if a pool becomes imbalanced. For instance, the function could allow the caller to swap the withdrawn tokens in external DEXs within the `restoreVault` function to achieve the most optimal proportion to minimize the penalty and slippage when re-entering the pool. This is similar to the approach in the vault's reinvest function.