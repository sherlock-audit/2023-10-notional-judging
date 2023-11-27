Harsh Heather Elk

medium

# Weighted pool spot price calculation is incorrect

## Summary
Notional calculate the spot price of Weighted pool using the balances of token. Which need to be upscaled according to the balancer but Notional doesn't.

## Vulnerability Detail
Weighted pool

```solidity
File: leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol
19    function getWeightedSpotPrices(
20      bytes32 poolId,
21        address poolAddress,
22        uint256 primaryIndex,
23        uint8 primaryDecimals
24    ) external view returns (uint256[] memory balances, uint256[] memory spotPrices) {
25        (/* */, balances, /* */) = Deployments.BALANCER_VAULT.getPoolTokens(poolId);
26        // Only two token pools are supported
27        require(balances.length == 2);
28        spotPrices = new uint256[](2);
29
30        uint256[] memory weights = IWeightedPool(poolAddress).getNormalizedWeights();
31
32        // Spot price calculation is specified at the link below. Do not account for swap fees
33        // because we're using this price to compare to the oracle price and adding swap fees
34       // would unnecessarily increase the price deviation.
35        // https://docs.balancer.fi/reference/math/weighted-math.html#typescript
36        // secondaryBalance * primaryWeight * primaryDecimals 
37        // --------------------------------------------------- 
38        //          primaryBalance * secondaryWeight
39        uint256 secondaryIndex = 1 - primaryIndex;
40
41        // There is a chance of a uint256 overflow if the balances[secondaryIndex] > 10**36
42        uint256 numerator = balances[secondaryIndex] * weights[primaryIndex] * (10 ** primaryDecimals);
43        uint256 denominator = balances[primaryIndex] * weights[secondaryIndex];
44        spotPrices[secondaryIndex] = numerator / denominator; //@audit-issue here we should have upscaled spot price (in 18 decimals)
45    }
```
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L19
 - [getWeightedSpotPrices](https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L19) is called to find the spot price
 - it fetches the balances and Normalized weight of token at Line 25 and 30
 - Then calculated the spot price -`Price at which first token can be exchanged for second token` without upscaling them
 - Since the pool hooks in balancer always work with upscaled balance so we need to manually upscale here for consistency [Reference](https://github.com/balancer/balancer-v2-monorepo/blob/master/pkg/pool-weighted/contracts/BaseWeightedPool.sol#L93) before calculating any params for WeighedMath

while 
```solidity
File:pkg/pool-weighted/contracts/BaseWeightedPool.sol
    function getInvariant() public view returns (uint256) {
        (, uint256[] memory balances, ) = getVault().getPoolTokens(getPoolId());

        // Since the Pool hooks always work with upscaled balances, we manually
        // upscale here for consistency
        _upscaleArray(balances, _scalingFactors()); //@audit upscale needed like this in notional

        uint256[] memory normalizedWeights = _getNormalizedWeights();
        return WeightedMath._calculateInvariant(normalizedWeights, balances);
    }
```
https://github.com/balancer/balancer-v2-monorepo/blob/master/pkg/pool-weighted/contracts/BaseWeightedPool.sol#L93
balancer recommend to upscale the balances for the consistence calculation throughout the WeightedMath

Hence this return unscaled spot price which can create further integration risk when decimals of tokens aren't same and not 18.
## Impact
Notional returns unscaled spot price for Weighted Pool which further leads to integration and an array of issues which is dependent on the spot price of Weighted Pool
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L19
https://github.com/balancer/balancer-v2-monorepo/blob/master/pkg/pool-weighted/contracts/BaseWeightedPool.sol#L93

## Tool used

Manual Review

## Recommendation
return upscaled spot price in Weighted pool like the composable pool