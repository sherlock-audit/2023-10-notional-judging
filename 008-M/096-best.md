Little Midnight Mockingbird

high

# Potential Permanent Locking of LP in Vaults Due to lock state Deadlock when `emergencyExit` executed

## Summary
 
Potential Permanent Locking of LP in Vaults Due to state Deadlock when `emergencyExit` executed before Curve pool is killed

## Vulnerability Detail

Curve pool have a kill switch for deactivate / kill a pool when it contains bugs or vulnerability. For a reference,
https://decrypt.co/56948/curve-finance-shuts-down-yv2-pool-after-finding-vulnerability

Once a curve pool is killed, it can only be accessed for withdrawal, but no deposit or else.

In Notional abstract contract `SingleSidedLPVaultBase`, there is `emergencyExit` function callable by `EMERGENCY_EXIT_ROLE` being use for any emergency situation. If this function is called, the vault will be locked.

```js
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
490: 
491:         state.totalPoolClaim = state.totalPoolClaim - claimToExit;
492:         state.setStrategyVaultState();
493: 
494:         emit EmergencyExit(claimToExit);
495:         _lockVault();
496:     }
```

If we take a look more into the function, it will call `_unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), true);`. This means it will unstake with `isSingleSided` to true. The call will then redeem a single sided `remove_liquidity_one_coin`.

```js
File: Curve2TokenConvexVault.sol
66:     function _unstakeAndExitPool(
67:         uint256 poolClaim, uint256[] memory _minAmounts, bool isSingleSided
68:     ) internal override returns (uint256[] memory exitBalances) {
...
77: 
78:         ICurve2TokenPool pool = ICurve2TokenPool(CURVE_POOL);
79:         exitBalances = new uint256[](2);
80:         if (isSingleSided) {
81:             // Redeem single-sided
82:             exitBalances[_PRIMARY_INDEX] = pool.remove_liquidity_one_coin(
83:                 poolClaim, int8(_PRIMARY_INDEX), _minAmounts[_PRIMARY_INDEX]
84:             );
85:         } else {
86:             // Redeem proportionally, min amounts are rewritten to a fixed length array
...
94:         }
95:     }
```

Looking into curve pool contract, there is a check inside `remove_liquidity_one_coin` where the pool should not be killed. 

```vy
@external
@nonreentrant('lock')
def remove_liquidity_one_coin(_token_amount: uint256, i: int128, _min_amount: uint256) -> uint256:
    """
    @notice Withdraw a single coin from the pool
    @param _token_amount Amount of LP tokens to burn in the withdrawal
    @param i Index value of the coin to withdraw
    @param _min_amount Minimum amount of coin to receive
    @return Amount of coin received
    """
    assert not self.is_killed  # dev: is killed
```

When `emergencyExit` is aim to be called due to a Curve pool contains a vulnerability thus killed, the `emergencyExit` calls will always reverted. Therefore, Notional vaults can't be locked if the Curve vault was killed.  Even if the vault is not locked, Notional can still redeem with proportional exit, but still normally the Vault need to be locked.

When vault in locked state, there is a function called `restoreVault` to unlock the vault and resume everything, thus `emergencyExit` is not a final exit, means, it's possible for a temporary lock / exit.

Now, if we consider a scenario, where:
- the `EMERGENCY_EXIT_ROLE` calls `emergencyExit` for a temporary lock, with `claimToExit` != 0, (there is still LP in vault)
- then, the Curve pool is killed

This condition will make the LP in vault is being locked permanently.

Because redeeming is not possible when vault is locked, and, to unlock the vault, the `restoreVault` should be called but eventually will reverted on `_joinPoolAndStake` (revert on `add_liquidity` due pool was killed) 

Since the toggle of lock and unlock of vault can be change through `emergencyExit` and `restoreVault`, even Notional owner can't do much about it.

## Impact

LP can be locked in vault, and redeeming is impossible due to deadlock on toggling the lock status flag

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L480-L496

## Tool used

Manual Review

## Recommendation

Consider to add another owner function to lock/unlock the vault other than `emergencyExit` and `restoreVault`