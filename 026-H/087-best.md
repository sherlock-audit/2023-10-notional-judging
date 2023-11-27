Shallow Peanut Elk

high

# Single-sided instead of proportional exit is performed during emergency exit

## Summary

Single-sided instead of proportional exit is performed during emergency exit, which could lead to a loss of assets during emergency exit and vault restoration.

## Vulnerability Detail

Per the comment in Line 476 below, the BPT should be redeemed proportionally to underlying tokens during an emergency exit. However, it was found that the `_unstakeAndExitPool` function is executed with the `isSingleSided` parameter set to `true`.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L480

```solidity
File: SingleSidedLPVaultBase.sol
475:     /// @notice Allows the emergency exit role to trigger an emergency exit on the vault.
476:     /// In this situation, the `claimToExit` is withdrawn proportionally to the underlying
477:     /// tokens and held on the vault. The vault is locked so that no entries, exits or
478:     /// valuations of vaultShares can be performed.
479:     /// @param claimToExit if this is set to zero, the entire pool claim is withdrawn
480:     function emergencyExit(
481:         uint256 claimToExit, bytes calldata /* data */
482:     ) external override onlyRole(EMERGENCY_EXIT_ROLE) {
483:         StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
484:         if (claimToExit == 0) claimToExit = state.totalPoolClaim;
485: 
486:         // By setting min amounts to zero, we will accept whatever tokens come from the pool
487:         // in a proportional exit. Front running will not have an effect since no trading will
488:         // occur during a proportional exit.
489:         _unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), true);
```

If the `isSingleSided` is set to `True`, the `EXACT_BPT_IN_FOR_ONE_TOKEN_OUT` will be used, which is incorrect. Per the Balancer's [documentation](https://docs.balancer.fi/reference/joins-and-exits/pool-exits.html#userdata), `EXACT_BPT_IN_FOR_ONE_TOKEN_OUT` is a single asset exit where the user sends a precise quantity of BPT, and receives an estimated but unknown (computed at run time) quantity of a single token.

To perform a proportional exit, the `EXACT_BPT_IN_FOR_TOKENS_OUT` should be used instead.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L67

```solidity
File: BalancerComposableAuraVault.sol
60:     function _unstakeAndExitPool(
61:         uint256 poolClaim, uint256[] memory minAmounts, bool isSingleSided
62:     ) internal override returns (uint256[] memory exitBalances) {
63:         bool success = AURA_REWARD_POOL.withdrawAndUnwrap(poolClaim, false); // claimRewards = false
64:         require(success);
65: 
66:         bytes memory customData;
67:         if (isSingleSided) {
..SNIP..
74:             uint256 primaryIndex = PRIMARY_INDEX();
75:             customData = abi.encode(
76:                 IBalancerVault.ComposableExitKind.EXACT_BPT_IN_FOR_ONE_TOKEN_OUT,
77:                 poolClaim,
78:                 primaryIndex < BPT_INDEX ?  primaryIndex : primaryIndex - 1
79:             );
```

The same issue affects the Curve's implementation of the `_unstakeAndExitPool` function.

```solidity
File: Curve2TokenConvexVault.sol
66:     function _unstakeAndExitPool(
67:         uint256 poolClaim, uint256[] memory _minAmounts, bool isSingleSided
68:     ) internal override returns (uint256[] memory exitBalances) {
..SNIP..
78:         ICurve2TokenPool pool = ICurve2TokenPool(CURVE_POOL);
79:         exitBalances = new uint256[](2);
80:         if (isSingleSided) {
81:             // Redeem single-sided
82:             exitBalances[_PRIMARY_INDEX] = pool.remove_liquidity_one_coin(
83:                 poolClaim, int8(_PRIMARY_INDEX), _minAmounts[_PRIMARY_INDEX]
84:             );
```

## Impact

The following are some of the impacts of this issue, which lead to loss of assets:

1. Redeeming LP tokens one-sided incurs unnecessary slippage as tokens have to be swapped internally to one specific token within the pool, resulting in fewer assets received.

2. Per the source code comment below, in other words, unless a proportional exit is performed, the emergency exit will be subjected to front-run attack and slippage.

   ```solidity
   File: SingleSidedLPVaultBase.sol
   486:         // By setting min amounts to zero, we will accept whatever tokens come from the pool
   487:         // in a proportional exit. Front running will not have an effect since no trading will
   488:         // occur during a proportional exit.
   489:         _unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), true);
   ```

3. After the emergency exit, the vault only held one of the pool tokens. To re-enter the pool, the vault has to either swap the token to other pool tokens on external DEXs to maintain the proportion or perform a single-sided join. Both of these methods will incur unnecessary slippage, resulting in fewer LP tokens received at the end.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L480

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L67

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L80

## Tool used

Manual Review

## Recommendation

Set the `isSingleSided` parameter to `false` when calling the `_unstakeAndExitPool` function to ensure that the proportional exit is performed.

```diff
function emergencyExit(
    uint256 claimToExit, bytes calldata /* data */
) external override onlyRole(EMERGENCY_EXIT_ROLE) {
    StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
    if (claimToExit == 0) claimToExit = state.totalPoolClaim;
	..SNIP..
-   _unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), true);
+   _unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), false);
```