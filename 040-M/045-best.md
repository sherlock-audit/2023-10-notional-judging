Cold Carob Cottonmouth

high

# CrossCurrencyVault.sol#_redeemfCash: IWrappedfCash#redeem function call will revert.

## Summary
Since the parameter types and numbers of `IWrappedfCash#redeem` function call in `CrossCurrencyVault.sol#_redeemfCash` does not coincide with function declaration, function calls will revert.

## Vulnerability Detail
`CrossCurrencyVault.sol#_redeemfCash` function is as follows.
```solidity
File: CrossCurrencyVault.sol
278:     function _redeemfCash(bool isETH, uint256 maturity, uint256 vaultShares) private {
279:         IWrappedfCash wfCash = getWrappedFCashAddress(maturity);
280:         uint256 assets = wfCash.redeem(vaultShares, address(this), address(this));
281: 
282:         if (isETH) WETH.withdraw(assets);
283:     }
```
On the other hand, `IWrappedfCash.sol#redeem` function declaration is as follows.
```solidity
File: IWrappedfCash.sol
34:     function redeem(uint256 amount, RedeemOpts memory data) external;
35:     function redeemToAsset(uint256 amount, address receiver, uint32 maxImpliedRate) external;
36:     function redeemToUnderlying(uint256 amount, address receiver, uint32 maxImpliedRate) external;
```
Since the parameter types and numbers of `IWrappedfCash#redeem` function call does not coincide with function declaration, function calls will revert.


## Impact
When `maturity < Constants.PRIME_CASH_VAULT_MATURITY`, `redeemFromNotional` and `convertVaultSharesToPrimeMaturity` function call to `CrossCurrencyVault` will revert at any time.

## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyVault.sol#L280

## Tool used

Manual Review

## Recommendation
Ensures that the parameter types and numbers of `IWrappedfCash#redeem` function call coincides with function declaration.