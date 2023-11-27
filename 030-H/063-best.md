Huge Cinnamon Dalmatian

high

# Vault will misprice the lp tokens when the underlying curve/balancer pool is imbalanced

## Summary
The calculation of the vaults' share token value in the primary token relies on Chainlink price feeds. Additionally, Chainlink price feeds are utilized to compare spot prices directly sourced from the pool. However, Chainlink operates independently from the pool. When computing the shares, the pool examines whether the prices align to detect potential manipulation, yet ultimately relies on the Chainlink oracle. This setup presents a concern as Chainlink's price data is entirely outsourced. Chainlink's price data should be used to detect potential manipulation, but it should not be the sole source for pricing the LP tokens to ensure more accurate pricing. 
## Vulnerability Detail
First, let's see how chainlink price and spot price is used to determine the value of LP token.
```solidity
                uint256 price = _getOraclePairPrice(primaryToken, address(tokens[i]));

                // Check that the spot price and the oracle price are near each other. If this is
                // not true then we assume that the LP pool is being manipulated.
                uint256 lowerLimit = price * (Constants.VAULT_PERCENT_BASIS - limit) / Constants.VAULT_PERCENT_BASIS;
                uint256 upperLimit = price * (Constants.VAULT_PERCENT_BASIS + limit) / Constants.VAULT_PERCENT_BASIS;
                if (spotPrices[i] < lowerLimit || upperLimit < spotPrices[i]) {
                    revert Errors.InvalidPrice(price, spotPrices[i]);
                }

                // Convert the token claim to primary using the oracle pair price.
                uint256 secondaryDecimals = 10 ** decimals[i];
                oneLPValueInPrimary += (tokenClaim * POOL_PRECISION() * primaryDecimals) / 
                    (price * secondaryDecimals);
```
As we can see in above snippet the actual pricing of the lp token is done with the chainlink price and spot price is only used to compare the chainlink price. The problem is the main price source is the spot price because it is directly calculating the price using the pool so the chainlink price should be used for comparison not for actual pricing. Now, let's examine some cases where this can be a problem:

Let's assume 3 different scenarios for a pool to see whether it makes sense to use chainlink price or the spot price. 
(p) = primary, (s) = secondary

1- Pool is balanced, 50-50:
This is the best case, using chainlink price or the spot price will have very less impact on valuing the LP since both sources will likely to return the same price. So we are estimating correctly here. 

2- Pool is %80 (p) - %20 (s) 
Here the spot price will actually give us that the 1 unit of secondary token is more valuable than 1 unit of primary token. So we can say the spot price is lesser than chainlink would give us considering chainlink fetches the data from plenty of markets. Using chainlink price to value the LP in this scenario will underprice the strategy tokens which is not what we expect because in real pool state the spot price will be used. When we actually single sided remove liquidity the spot price will be way more primary token then a 50-50 withdrawal. So we underestimate the primary tokens we get here.

3- Pool is %20 (p) - %80 (s)
Here the spot price will give us that 1 unit of secondary token is less valuable than 1 unit o primary token. When we actually do a single sided remove liquidity to primary token in this pool state we get lesser primary token. So we overestimate the primary tokens we get here. 

As we can say using spot price in all of these 3 scenarios are more realistic and accurate. However, we assumed that there is no actual bad action such as manipulating the pool prices although the balances are normal. Let's talk about that a bit.

Considering the above 3 scenarios especially when the pool is deviated we need a higher `oraclePriceDeviationLimitPercent` regardless of the price manipulation because the spot price and the chainlink price is not going to be close in such cases. However, when the pool is 50-50 having a high `oraclePriceDeviationLimitPercent` and picking the spot price might be problematic because someone can always manipulate the price since we will always pick the spot price. Well, here are the 2 approaches that can easily mitigate that

1- Admin checks the pool balances and if they are imbalanced admin sets the `oraclePriceDeviationLimitPercent` to a higher amount. Note that this has to be done in either you chose to use spot price or chainlink price. When the pool balances are balanced or near balanced admin sets this to a lesser value to ensure manipulation of the pool is not possible. 

2- Always pick the worst price that will give us the most pessimictic result in primary token for 1 LP. Pessimistic approach here actually is not a problem because liquidity pools are in its nature risky products. Also, this function is essentially used for valuing a vault token that user has which is used in liquidations so doing a pessimistic approach here can actually help but it can also allow bad liquidators to liquidate people leveraging this. 

In short, when the pool is imbalanced the valuing of LP tokens will be wrong. 

Here is the PoC test demonstrating how spot prices deviates
NOTE: amount is half of the token0 or token1 amount and lpAmount is 25% of the entire supply
```solidity
function test_getDy_balanced() public {
        deal(CURVE_POOL, TAPIR, lpAmount);
        assertEq(IERC20(CURVE_POOL).balanceOf(TAPIR), lpAmount);

        uint256 currentPrice = ICurvePool(CURVE_POOL).get_dy(0, 1, 1e18);
        console.log("Current price", currentPrice);

        uint bal0 = ICurvePool(CURVE_POOL).balances(0);
        uint bal1 = ICurvePool(CURVE_POOL).balances(1);
        uint ts = ICurvePool(CURVE_POOL).totalSupply();

        uint token0Claim = (lpAmount * bal0) / ts;
        uint token1Claim = (lpAmount * bal1) / ts;

        uint256 spotPriceToAmount = ((token1Claim * 1e18) / currentPrice);
        uint totalLPInPrimary = token0Claim + spotPriceToAmount;
        console.log("Spot price assumes the amount", totalLPInPrimary);

        hoax(TAPIR);
        uint256 actualRemovenLiquidity = ICurvePool(CURVE_POOL).remove_liquidity_one_coin(lpAmount, 0, 0);
        console.log("Actual amount if remove single sided liq happens", actualRemovenLiquidity);
    }

    function test_getDy_moreToken0() public {
        deal(CURVE_POOL, TAPIR, lpAmount);
        assertEq(IERC20(CURVE_POOL).balanceOf(TAPIR), lpAmount);

        hoax(TAPIR);
        ICurvePool(CURVE_POOL).exchange{value: amount}(0, 1, amount, 0);

        uint256 currentPrice = ICurvePool(CURVE_POOL).get_dy(0, 1, 1e18);
        console.log("Spot price", currentPrice);

        uint bal0 = ICurvePool(CURVE_POOL).balances(0);
        uint bal1 = ICurvePool(CURVE_POOL).balances(1);
        uint ts = ICurvePool(CURVE_POOL).totalSupply();

        uint token0Claim = (lpAmount * bal0) / ts;
        uint token1Claim = (lpAmount * bal1) / ts;

        uint256 spotPriceToAmount = ((token1Claim * 1e18) / currentPrice);
        uint totalLPInPrimary = token0Claim + spotPriceToAmount;
        console.log("Spot price assumes the amount", totalLPInPrimary);

        hoax(TAPIR);
        uint256 actualRemovenLiquidity = ICurvePool(CURVE_POOL).remove_liquidity_one_coin(lpAmount, 0, 0);
        console.log("Actual amount if remove single sided liq happens", actualRemovenLiquidity);
    }

    function test_getDy_moreToken1() public
``` 

## Impact
Using the spot price always makes more sense. If the pool is imbalanced we need a bigger `oraclePriceDeviationLimitPercent` anyways so it is not safer to use chainlink price. Spot price should be used because it is the real liquidity value that the vault is handling and since in pool imbalances this will not be accurate and uses chainlink the valuing lp will be wrong and get lead to unfair liquidations hence, I will label this as high
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L333-L367
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L97-L114

## Tool used

Manual Review

## Recommendation
Check the vulnerability details last sections. In short, `oraclePriceDeviationLimitPercent` requires active management from Notional admin anyways so no need to choose the chainlink price over spot price. Or alternatively you can use the pessimistic  value comparing the two oracles but I am not sure how the valuing of vault tokens used across the Notional ecosystem so I would say go with the 1 option.