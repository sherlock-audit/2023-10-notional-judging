Shallow Peanut Elk

high

# Different spot prices used during the comparison

## Summary

The spot prices used during the comparison are different, which might result in the trade proceeding even if the pool is manipulated, leading to a loss of assets.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L90

```solidity
File: BalancerComposableAuraVault.sol
090:     function _checkPriceAndCalculateValue() internal view override returns (uint256) {
091:         (uint256[] memory balances, uint256[] memory spotPrices) = SPOT_PRICE.getComposableSpotPrices(
092:             BALANCER_POOL_ID,
093:             address(BALANCER_POOL_TOKEN),
094:             PRIMARY_INDEX()
095:         );
096: 
097:         // Spot prices are returned in native decimals, convert them all to POOL_PRECISION
098:         // as required in the _calculateLPTokenValue method.
099:         (/* */, uint8[] memory decimals) = TOKENS();
100:         for (uint256 i; i < spotPrices.length; i++) {
101:             spotPrices[i] = spotPrices[i] * POOL_PRECISION() / 10 ** decimals[i];
102:         }
103: 
104:         return _calculateLPTokenValue(balances, spotPrices);
105:     }
```

Line 91 above calls the `SPOT_PRICE.getComposableSpotPrices` function to fetch the spot prices. Within the function, it relies on the `StableMath._calcSpotPrice` function to compute the spot price. Per the comments of this function, `spot price of token Y in token X` and `spot price Y/X` means that the Y (base) / X (quote). Thus, secondary (base) / primary (quote).

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/math/StableMath.sol#L90

```solidity
File: StableMath.sol
087:     /**
088:      * @dev Calculates the spot price of token Y in token X.
089:      */
090:     function _calcSpotPrice(
091:         uint256 amplificationParameter,
092:         uint256 invariant, 
093:         uint256 balanceX,
094:         uint256 balanceY
095:     ) internal pure returns (uint256) {
096:         /**************************************************************************************************************
097:         //                                                                                                           //
098:         //                             2.a.x.y + a.y^2 + b.y                                                         //
099:         // spot price Y/X = - dx/dy = -----------------------                                                        //
100:         //                             2.a.x.y + a.x^2 + b.x                                                         //
101:         //                                                                                                           //
102:         // n = 2                                                                                                     //
103:         // a = amp param * n                                                                                         //
104:         // b = D + a.(S - D)                                                                                         //
105:         // D = invariant                                                                                             //
106:         // S = sum of balances but x,y = 0 since x  and y are the only tokens                                        //
107:         **************************************************************************************************************/
108: 
109:         unchecked {
110:             uint256 a = (amplificationParameter * 2) / _AMP_PRECISION;
```

The above spot price will be used within the `_calculateLPTokenValue` function to compare with the oracle price to detect any potential pool manipulation. However, the oracle price returned is in primary (base) / secondary (quote) format. As such, the comparison between the spot price (secondary-base/primary-quote) and oracle price (primary-base/secondary-quote) will be incorrect.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L355

```solidity
File: SingleSidedLPVaultBase.sol
333:     function _calculateLPTokenValue(
..SNIP..
351:                 uint256 price = _getOraclePairPrice(primaryToken, address(tokens[i]));
352: 
353:                 // Check that the spot price and the oracle price are near each other. If this is
354:                 // not true then we assume that the LP pool is being manipulated.
355:                 uint256 lowerLimit = price * (Constants.VAULT_PERCENT_BASIS - limit) / Constants.VAULT_PERCENT_BASIS;
356:                 uint256 upperLimit = price * (Constants.VAULT_PERCENT_BASIS + limit) / Constants.VAULT_PERCENT_BASIS;
357:                 if (spotPrices[i] < lowerLimit || upperLimit < spotPrices[i]) {
358:                     revert Errors.InvalidPrice(price, spotPrices[i]);
359:                 }
```

## Impact

If the spot price is incorrect, it might potentially fail to detect the pool has been manipulated or result in unintended reverts due to false positives. In the worst-case scenario, the trade proceeds to execute against the manipulated pool, leading to a loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L90

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/math/StableMath.sol#L90

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L355

## Tool used

Manual Review

## Recommendation

Consider verifying if the comment of the `StableMath._calcSpotPrice` function is aligned with its implementation with the Balancer team. 

In addition, the `StableMath._calcSpotPrice` function is no longer used or found within the current version of Balancer's composable pool. Thus, there is no guarantee that the math within the `StableMath._calcSpotPrice` works with the current implementation. It is recommended to use the existing method in the current Composable Pool's StableMath, such as `_calcOutGivenIn` (ensure the fee is excluded) to compute the spot price.