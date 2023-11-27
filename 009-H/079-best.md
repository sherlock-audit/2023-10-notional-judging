Shallow Peanut Elk

high

# Incorrect scaling of the spot price

## Summary

The incorrect scaling of the spot price leads to the incorrect spot price, which is later compared with the oracle price.

If the spot price is incorrect, it might potentially fail to detect the pool has been manipulated or result in unintended reverts due to false positives. In the worst-case scenario, the trade proceeds to execute against the manipulated pool, leading to a loss of assets.

## Vulnerability Detail

Per the comment and source code at Lines 97 to 103, the `SPOT_PRICE.getComposableSpotPrices` is expected to return the spot price in native decimals.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L97

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

Within the `getComposableSpotPrices` function, it will trigger the `_calculateStableMathSpotPrice` function. When the primary and secondary balances are passed into the `StableMath._calculateInvariant` and `StableMath._calcSpotPrice` functions, they are scaled up to 18 decimals precision as StableMath functions only work with balances that have been normalized to 18 decimals.

Assuming that the following states:

- Primary Token = USDC (6 decimals)
- Secondary Token = DAI (18 decimals)
- Primary Balance = 100 USDC (=100 * 1e6)
- Secondary Balance = 100 DAI (=100 * 1e18)
- scalingFactors[USDC] = 1e12 * Fixed.ONE (1e18) = 1e30
- scalingFactors[DAI] = 1e0 * Fixed.ONE (1e18) = 1e18
- The price between USDC and DAI is 1:1

After scaling the primary and secondary balances, the scaled balances will be as follows:

```solidity
scaledPrimary = balances[USDC] * scalingFactors[USDC] / BALANCER_PRECISION
scaledPrimary = 100 * 1e6 * 1e30 / 1e18
scaledPrimary = 100 * 1e18

scaledSecondary = balances[DAI] * scalingFactors[DAI] / BALANCER_PRECISION
scaledSecondary = 100 * 1e18 * 1e18 / 1e18
scaledSecondary = 100 * 1e18
```

The spot price returned from the `StableMath._calcSpotPrice` function at Line 93 will be `1e18` (1:1).

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L93

```solidity
File: BalancerSpotPrice.sol
78:     function _calculateStableMathSpotPrice(
79:         uint256 ampParam,
80:         uint256[] memory scalingFactors,
81:         uint256[] memory balances,
82:         uint256 scaledPrimary,
83:         uint256 primaryIndex,
84:         uint256 index2
85:     ) internal pure returns (uint256 spotPrice) {
86:         // Apply scale factors
87:         uint256 secondary = balances[index2] * scalingFactors[index2] / BALANCER_PRECISION;
88: 
89:         uint256 invariant = StableMath._calculateInvariant(
90:             ampParam, StableMath._balances(scaledPrimary, secondary), true // round up
91:         );
92: 
93:         spotPrice = StableMath._calcSpotPrice(ampParam, invariant, scaledPrimary, secondary);
94: 
95:         // Remove scaling factors from spot price
96:         spotPrice = spotPrice * scalingFactors[primaryIndex] / scalingFactors[index2];
97:     }
```

Subsequently, in Line 96 above, the code attempts to remove the scaling factor from the spot price (1e18).

```solidity
spotPrice = spotPrice * scalingFactors[USDC] / scalingFactors[DAI];
spotPrice = 1e18 * 1e30 / 1e18
spotPrice = 1e30
spotPrice = 1e12 * 1e18
```

The `spotPrice[DAI-Secondary]` is not denominated in native precision after the scaling. The `SPOT_PRICE.getComposableSpotPrices` will return the following spot prices:

```solidity
spotPrice[USDC-Primary] = 0
spotPrice[DAI-Secondary] = 1e12 * 1e18
```

The returned spot prices will be scaled to POOL_PRECISION (1e18). After the scaling, the spot price remains the same:

```solidity
spotPrice[DAI-Secondary] = spotPrice[DAI-Secondary] * POOL_PRECISION / DAI_Decimal
spotPrice[DAI-Secondary] = 1e12 * 1e18 * 1e18 / 1e18
spotPrice[DAI-Secondary] = 1e12 * 1e18
```

The converted spot prices will be passed into the `_calculateLPTokenValue` function. Within the `_calculateLPTokenValue` function, the oracle price for DAI<>USDC will be `1e18`. From here, the `spotPrice[DAI-Secondary]` (1e12 * 1e18) is significantly different from the oracle price (1e18), which will cause the [pool manipulation check](https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L353) to revert.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L97

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

## Impact

The spot price is used to verify if the pool has been manipulated before executing certain key vault actions (e.g. reinvest rewards). 

If the spot price is incorrect, it might potentially result in the following:

- Failure to detect the pool has been manipulated, resulting in the trade to execute against the manipulated pool, leading to a loss of assets.
- Unintended reverts due to false positives, breaking core functionalities of the protocol that rely on the `_checkPriceAndCalculateValue` function.

The affected `_checkPriceAndCalculateValue` function was found to be used within the following functions:

- `reinvestReward` - If the `_checkPriceAndCalculateValue` function is malfunctioning or reverts unexpectedly, the protocol will not be able to reinvest, leading to a loss of value for the vault shareholders.

- `convertStrategyToUnderlying` - This function is used by Notional V3 for the purpose of computing the collateral values and the account's health factor. If the `_checkPriceAndCalculateValue` function reverts unexpectedly due to an incorrect invariant/spot price, many of Notional's core functions will break. In addition, the collateral values and the account's health factor might be inflated if it fails to detect a manipulated pool due to incorrect invariant/spot price, potentially allowing the malicious actors to drain the main protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L97

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L93

## Tool used

Manual Review

## Recommendation

The spot price returned from `StableMath._calcSpotPrice` is denominated in 1e18 (POOL_PRECISION) since the inputted balances are normalized to 18 decimals. The scaling factors are used to normalize a balance to 18 decimals. By dividing or scaling down the spot price by the scaling factor, the native spot price will be returned.

```solidity
spotPrice[DAI-Secondary] = spotPrice[DAI-Secondary] * Fixed.ONE / scalingFactors[DAI];
spotPrice = 1e18 * Fixed.ONE / (1e0 * Fixed.ONE)
spotPrice = 1e18 * 1e18 / (1e0 * 1e18)
spotPrice = 1e18
```