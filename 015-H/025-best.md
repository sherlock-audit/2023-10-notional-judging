Festive Misty Dog

high

# Funds Loss Risk in Exchange Rate Calculation

## Summary
The `getExchangeRate()` function poses a risk of funds loss within the protocol, as the current price of a vault share may be overestimated, leading to potential financial discrepancies during exchanges.

## Vulnerability Detail
The function `Curve2TokenConvexVault._checkPriceAndCalculateValue()` fetches balances from `CURVE_POOL` using `CURVE_POOL.balances(0)` and `CURVE_POOL.balances(1)`, then uses `get_dy` to obtain the price of one unit of the primary token converted to the secondary token.
```solidity
    function _checkPriceAndCalculateValue() internal view override returns (uint256 oneLPValueInPrimary) {
        uint256[] memory balances = new uint256[](2);
        balances[0] = ICurvePool(CURVE_POOL).balances(0);
        balances[1] = ICurvePool(CURVE_POOL).balances(1);

        // The primary index spot price is left as zero.
        uint256[] memory spotPrices = new uint256[](2);
        uint256 primaryPrecision = 10 ** PRIMARY_DECIMALS;
        uint256 secondaryPrecision = 10 ** SECONDARY_DECIMALS;

        // `get_dy` returns the price of one unit of the primary token
        // converted to the secondary token. The spot price is in secondary
        // precision and then we convert it to POOL_PRECISION.
        spotPrices[SECONDARY_INDEX] = ICurvePool(CURVE_POOL).get_dy(
            int8(_PRIMARY_INDEX), int8(SECONDARY_INDEX), primaryPrecision
        ) * POOL_PRECISION() / secondaryPrecision;

        return _calculateLPTokenValue(balances, spotPrices);
    }

```
 Finally, the function invokes an internal function `_calculateLPTokenValue()` to determine the value of one LP token based on the obtained balances and spot prices from the Curve pool.   The `oneLPValueInPrimary` is calculated as follows.

```solidity
  uint256 tokenClaim = balances[i] * POOL_PRECISION() / totalSupply;
            if (i == PRIMARY_INDEX()) {
                oneLPValueInPrimary += tokenClaim;
            } else {
......
 oneLPValueInPrimary += (tokenClaim * POOL_PRECISION() * primaryDecimals) / 
                    (price * secondaryDecimals);
}

```


However, the issue arises from the fact that `CurvePool(CURVE_POOL).balances()` can be manipulated by malicious actors.

Bad actors could potentially deposit funds into CURVE_POOL in advance, artificially inflating the balances. As a result, when calculating the LP token value, the protocol would derive a value greater than expected due to the manipulated balances.

Therefore, in the `getExchangeRate()` function, the current price of a vault share will be greater than expected, as indicated in the following code snippet.
```solidity
function getExchangeRate(uint256 /* maturity */) external view override returns (int256) {
        StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
        uint256 oneLPValueInPrimary = _checkPriceAndCalculateValue();
        // If inside an emergency exit, just report the one LP value in primary since the total
        // pool claim will be 0
        if (state.totalVaultSharesGlobal == 0 || isLocked()) {
            return oneLPValueInPrimary.toInt();
        } else {
            uint256 lpTokensPerVaultShare = (uint256(Constants.INTERNAL_TOKEN_PRECISION) * state.totalPoolClaim)
                / state.totalVaultSharesGlobal;
            return (oneLPValueInPrimary * lpTokensPerVaultShare / POOL_PRECISION()).toInt();
        }
    }

```
Furthermore, in the `reinvestReward()` function, the invocation of `_checkPriceAndCalculateValue()` may lead to a revert, preventing the successful execution of the reinvestment process, possibly due to the the same issue.

## Impact
Resulting in a loss of funds within the protocol during exchanges.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L99-L100
## Tool used

Manual Review

## Recommendation
