Shallow Peanut Elk

high

# Incorrect invariant used for Balancer's composable pools

## Summary

Only two balances instead of all balances were used when computing the invariant for Balancer's composable pools, which is incorrect. As a result, pool manipulation might not be detected. This could lead to the transaction being executed on the manipulated pool, resulting in a loss of assets.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L90

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
..SNIP..
```

A composable pool can support up to 5 tokens (excluding the BPT). When computing the invariant for a composable pool, one needs to pass in the balances of all the tokens within the pool except for BPT. However, the existing code always only passes in the balance of two tokens, which will return an incorrect invariant if the composable pool supports more than two tokens.

Following is the formula for computing the invariant of a composable pool taken from Balancer's Composable Pool. The `balances` passed into this function consist of all balances except for BPT ([Reference](https://github.com/balancer/balancer-v2-monorepo/blob/6da0b87fa7bd10cbba7f084d45336fc743944fea/pkg/pool-stable/contracts/ComposableStablePool.sol#L700))

https://github.com/balancer/balancer-v2-monorepo/blob/c7d4abbea39834e7778f9ff7999aaceb4e8aa048/pkg/pool-stable/contracts/StableMath.sol#L57

```solidity
function _calculateInvariant(uint256 amplificationParameter, uint256[] memory balances)
    internal
    pure
    returns (uint256)
{
    /**********************************************************************************************
    // invariant                                                                                 //
    // D = invariant                                                  D^(n+1)                    //
    // A = amplification coefficient      A  n^n S + D = A D n^n + -----------                   //
    // S = sum of balances                                             n^n P                     //
    // P = product of balances                                                                   //
    // n = number of tokens                                                                      //
    **********************************************************************************************/
```

The Balancer SDK's provide a feature to compute the spot price of any two tokens within a pool (https://github.com/balancer/balancer-sdk/blob/develop/balancer-js/src/modules/pools/pool-types/concerns/stablePhantom/spotPrice.spec.ts). By tracing the functions, it eventually triggers the following `_poolDerivatives` function.

Within the `_poolDerivatives` function, the `balances` used to compute the invariant consist of the balance of all tokens in the pool, except for BPT, which is aligned with the earlier understanding.

https://github.com/balancer/balancer-sor/blob/73d6b435c1429bbfc199b39b38a36e581838d2c3/src/pools/phantomStablePool/phantomStableMath.ts#L516

```solidity
export function _poolDerivatives(
    A: BigNumber,
    balances: OldBigNumber[],
    tokenIndexIn: number,
    tokenIndexOut: number,
    is_first_derivative: boolean,
    wrt_out: boolean
): OldBigNumber {
    const totalCoins = balances.length;
    const D = _invariant(A, balances);
```

Note: Composable Pool used to be called Phantom Pool in the past (https://medium.com/balancer-protocol/rate-manipulation-in-balancer-boosted-pools-technical-postmortem-53db4b642492)

## Impact

An incorrect invariant will lead to an incorrect spot price being computed. The spot price is used within the `_checkPriceAndCalculateValue` function that is intended to revert if the spot price on the pool is not within some deviation tolerance of the implied oracle price to prevent any pool manipulation. As a result, incorrect spot price leads to false positives or false negatives, where, in the worst-case scenario, pool manipulation was not caught by this function, and the transaction proceeded to be executed. 

The `_checkPriceAndCalculateValue` function was found to be used within the following functions:

- `reinvestReward` - If the `_checkPriceAndCalculateValue` function is malfunctioning, it will cause the vault to add liquidity into a pool that has been manipulated, leading to a loss of assets.

- `convertStrategyToUnderlying` - This function is used by Notional V3 for the purpose of computing the collateral values and the account's health factor. If the `_checkPriceAndCalculateValue` function reverts unexpectedly due to an incorrect invariant/spot price, many of Notional's core functions will break. In addition, the collateral values and the account's health factor might be inflated if it fails to detect a manipulated pool due to incorrect invariant/spot price, potentially allowing the malicious actors to drain the main protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L90

## Tool used

Manual Review

## Recommendation

Review if there is any specific reason for passing in only the balance of two tokens when computing the invariant. Otherwise, the balance of all tokens (except BPT) should be used to compute the invariant. 

In addition, it is recommended to include additional tests to ensure that the computed spot price is aligned with the market price.