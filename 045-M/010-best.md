Colossal Crepe Dove

medium

# The `convertVaultSharesToPrimeMaturity()` function lacks the whenNotLocked() modifier

## Summary
All key external functions in the vault are supposed to have the `whenNotLocked()` modifier, in case of an emergency but `convertVaultSharesToPrimeMaturity()` doesn't
## Vulnerability Detail
From the `convertVaultSharesToPrimeMaturity()` function an `account` can withdraw settled fCash to underlying from the fCash wrapper and deposit back into Notional as Prime Cash from `Notional` contract.

The issue is that this feature should also be restricted in the case of an emergency.

When there is an emergency the vault has to be fully locked, i.e no funds entering and no funds leaving the vaults.

Moreover the `SingleSidedLPVaultBase.emergencyExit()` will lock the vault so that no entries, exits or valuations of vaultShares can be performed, but since the `convertVaultSharesToPrimeMaturity()` function lacks the `whenNotLocked()` modifier, there will still be exits and valuations of vaultShares in the locked vault.

## Impact
An `account` can withdraw settled fCash to underlying from the fCash wrapper and deposit back into Notional as Prime Cash from `Notional` contract even when there's an emergency that requires the vaults to be fully locked.

The vault is locked so that no entries, exits or valuations of vaultShares can be performed but the intention for which the  vaults were locked can be ruined  via  `convertVaultSharesToPrimeMaturity()` function  as it lacks the  `whenNotLocked()` modifier.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/BaseStrategyVault.sol#L208
## Tool used

Manual Review

## Recommendation
Add the `whenNotLocked()` modifier to the `convertVaultSharesToPrimeMaturity()` function