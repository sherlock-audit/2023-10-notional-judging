Shallow Peanut Elk

medium

# Potential rounding errors during deposit and redemption

## Summary

Due to a rounding error in Solidity, it is possible that a user deposits assets to the vault but receives no vault shares in return OR redeems vault shares but does not receive any asset in return.

## Vulnerability Detail

Due to a rounding error in Solidity, it is possible that a user deposits assets to the vault but receives no value share in return due to issues in the following functions:

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L229

```solidity
File: SingleSidedLPVaultBase.sol
229:     function _mintVaultShares(uint256 lpTokens) internal returns (uint256 vaultShares) {
230:         StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
231:         if (state.totalPoolClaim == 0) {
232:             // Vault Shares are in 8 decimal precision
233:             vaultShares = (lpTokens * uint256(Constants.INTERNAL_TOKEN_PRECISION)) / POOL_PRECISION();
234:         } else {
235:             vaultShares = (lpTokens * state.totalVaultSharesGlobal) / state.totalPoolClaim;
236:         }
```

On the other hand, it is possible that the user redeems the vault share but receives no asset in return due to a rounding error.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L293

```solidity
File: SingleSidedLPVaultBase.sol
293:     function _redeemVaultShares(uint256 vaultShares) internal returns (uint256 poolClaim) {
294:         StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
295:         // Will revert on divide by zero, which is the correct behavior
296:         poolClaim = (vaultShares * state.totalPoolClaim) / state.totalVaultSharesGlobal;
297: 
298:         state.totalPoolClaim -= poolClaim;
299:         // Will revert on underflow if vault shares is greater than total shares global
300:         state.totalVaultSharesGlobal -= vaultShares.toUint80();
301:         state.setStrategyVaultState();
302:     }
```

The issue is similar to past contest issue (https://github.com/sherlock-audit/2022-12-notional-judging/issues/16)

## Impact

Loss of assets for the users as they deposited their assets but received zero vault shares in return OR they redeemed vault shares but did not receive any asset in return.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L229

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L293

## Tool used

Manual Review

## Recommendation

Consider reverting if no strategy token is minted during the deposit and no assets are returned during redemption.

If this issue has already been addressed by enforcing the minimum amount to prevent rounding to zero on the Notional V3 side as per the Discord message (https://discord.com/channels/812037309376495636/1175450365395751023/1177024379780083732), this issue can be ignored. 

However, care should be taken if there are other integrations with the leverage vault in the future that do not explicitly enforce such restrictions.