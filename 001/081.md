Shallow Peanut Elk

high

# Incorrect Spot Price

## Summary

Multiple discrepancies between the implementation of Leverage Vault's `_calcSpotPrice` function and SDK were observed, which indicate that the computed spot price is incorrect.

If the spot price is incorrect, it might potentially fail to detect the pool has been manipulated. In the worst-case scenario, the trade proceeds to execute against the manipulated pool, leading to a loss of assets.

## Vulnerability Detail

The `BalancerSpotPrice._calculateStableMathSpotPrice` function relies on the `StableMath._calcSpotPrice` to compute the spot price of two tokens.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L93

```solidity
File: BalancerSpotPrice.sol
78:     function _calculateStableMathSpotPrice(
..SNIP..
86:         // Apply scale factors
87:         uint256 secondary = balances[index2] * scalingFactors[index2] / BALANCER_PRECISION;
88: 
89:         uint256 invariant = StableMath._calculateInvariant(
90:             ampParam, StableMath._balances(scaledPrimary, secondary), true // round up
91:         );
92: 
93:         spotPrice = StableMath._calcSpotPrice(ampParam, invariant, scaledPrimary, secondary);
```

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
111:             uint256 b = Math.mul(invariant, a).sub(invariant);
112: 
113:             uint256 axy2 = Math.mul(a * 2, balanceX).mulDown(balanceY); // n = 2
114: 
115:             // dx = a.x.y.2 + a.y^2 - b.y
116:             uint256 derivativeX = axy2.add(Math.mul(a, balanceY).mulDown(balanceY)).sub(b.mulDown(balanceY));
117: 
118:             // dy = a.x.y.2 + a.x^2 - b.x
119:             uint256 derivativeY = axy2.add(Math.mul(a, balanceX).mulDown(balanceX)).sub(b.mulDown(balanceX));
120: 
121:             // The rounding direction is irrelevant as we're about to introduce a much larger error when converting to log
122:             // space. We use `divUp` as it prevents the result from being zero, which would make the logarithm revert. A
123:             // result of zero is therefore only possible with zero balances, which are prevented via other means.
124:             return derivativeX.divUp(derivativeY);
125:         }
126:     }
```

On a high level, the spot price is computed by determining the pool derivatives. The Balancer SDK's provide a feature to compute the [spot price of any two tokens](https://github.com/balancer/balancer-sdk/blob/develop/balancer-js/src/modules/pools/pool-types/concerns/stablePhantom/spotPrice.spec.ts) within a pool, and it leverages the [`_poolDerivatives`](https://github.com/balancer/balancer-sor/blob/73d6b435c1429bbfc199b39b38a36e581838d2c3/src/pools/phantomStablePool/phantomStableMath.ts#L507) function.

The existing function for computing the spot price of any two tokens of a composable pool has the following errors or discrepancies from the approach used to compute the spot price in Balancer SDK, which might lead to an inaccurate spot price being computed.

**Instance 1**

The comments and SDK add `b.y` and `b.x` to the numerator and denominator, respectively, in the formula. However, the code performs a subtraction.

**Instance  2**

Per the comment and SDK code, $b = (S - D) a + D$.

However, assuming that $S$ is zero (for a two-token pool), the following code in the Leverage Vault to compute $b$ is not equivalent to the above.

```solidity
uint256 b = Math.mul(invariant, a).sub(invariant);
```

**Instance 3**

The $S$ in the code will always be zero because the code is catered only for two-token pools. However, for a composable pool, it can support up to five (5) tokens in a pool. $S$ should be as follows, where $balances$ is all the tokens in a composable pool except for BPT.

$$
S = \sum_{i \neq \text{tokenIndexIn}, i \neq \text{tokenIndexOut}} \text{balances}[i]
$$

**Instance 4**

The amplification factor is scaled by  `A * 2` in the code, while the SDK scaled it by  `A * 2^2` (ATimesNpowN). https://github.com/balancer/balancer-sor/blob/73d6b435c1429bbfc199b39b38a36e581838d2c3/src/pools/stablePool/stableMath.ts#L235C63-L235C74

**Instance 5**

Per [SDK](https://github.com/balancer/balancer-sor/blob/73d6b435c1429bbfc199b39b38a36e581838d2c3/src/pools/stablePool/stableMath.ts#L364), the amplification factor is scaled down by $n^{(n - 1)}$ where $n$ is the number of tokens in a composable pool (excluding BPT).  Otherwise, this was not implemented within the code.

## Impact

The spot price is used to verify if the pool has been manipulated before executing certain key vault actions (e.g. reinvest rewards). If the spot price is incorrect, it might potentially fail to detect the pool has been manipulated or result in unintended reverts due to false positives. In the worst-case scenario, the trade proceeds to execute against the manipulated pool, leading to a loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L93

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/math/StableMath.sol#L90

## Tool used

Manual Review

## Recommendation

Given multiple discrepancies between the implementation of Leverage Vault's `_calcSpotPrice` function and SDK and due to the lack of information on the web, it is recommended to reach out to the Balancer's protocol team to identify the actual formula used to determine a spot price of any two tokens within a composable pool and check out if the formula in the SDK is up-to-date to be used against the composable pool.

It is also recommended to implement additional tests to ensure that the `_calcSpotPrice` returns the correct spot price of composable pools.

In addition, the `StableMath._calcSpotPrice` function is no longer used or found within the current version of Balancer's composable pool. Thus, there is no guarantee that the math within the `StableMath._calcSpotPrice` works with the current implementation. It is recommended to use the existing method in the current Composable Pool's StableMath, such as `_calcOutGivenIn` (ensure the fee is excluded) to compute the spot price.