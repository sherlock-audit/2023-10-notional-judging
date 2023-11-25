Shallow Peanut Elk

high

# Rounding differences when computing the invariant

## Summary

The invariant is used to compute the spot price to verify if the pool has been manipulated before executing certain key vault actions (e.g. reinvest rewards). If the inputted invariant is inaccurate, the spot price computed might not be accurate and might not match the actual spot price of the Balancer Pool. In the worst-case scenario, it might potentially fail to detect the pool has been manipulated, and the trade proceeds to execute against the manipulated pool, leading to a loss of assets.

## Vulnerability Detail

The Balancer's Composable Pool codebase relies on the old version of the `StableMath._calculateInvariant` that allows the caller to specify if the computation should round up or down via the `roundUp` parameter.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/math/StableMath.sol#L28

```python
File: StableMath.sol
28:     function _calculateInvariant(
29:         uint256 amplificationParameter,
30:         uint256[] memory balances,
31:         bool roundUp
32:     ) internal pure returns (uint256) {
33:         /**********************************************************************************************
34:         // invariant                                                                                 //
35:         // D = invariant                                                  D^(n+1)                    //
36:         // A = amplification coefficient      A  n^n S + D = A D n^n + -----------                   //
37:         // S = sum of balances                                             n^n P                     //
38:         // P = product of balances                                                                   //
39:         // n = number of tokens                                                                      //
40:         *********x************************************************************************************/
41: 
42:         unchecked {
43:             // We support rounding up or down.
44:             uint256 sum = 0;
45:             uint256 numTokens = balances.length;
46:             for (uint256 i = 0; i < numTokens; i++) {
47:                 sum = sum.add(balances[i]);
48:             }
49:             if (sum == 0) {
50:                 return 0;
51:             }
52: 
53:             uint256 prevInvariant = 0;
54:             uint256 invariant = sum;
55:             uint256 ampTimesTotal = amplificationParameter * numTokens;
56: 
57:             for (uint256 i = 0; i < 255; i++) {
58:                 uint256 P_D = balances[0] * numTokens;
59:                 for (uint256 j = 1; j < numTokens; j++) {
60:                     P_D = Math.div(Math.mul(Math.mul(P_D, balances[j]), numTokens), invariant, roundUp);
61:                 }
62:                 prevInvariant = invariant;
63:                 invariant = Math.div(
64:                     Math.mul(Math.mul(numTokens, invariant), invariant).add(
65:                         Math.div(Math.mul(Math.mul(ampTimesTotal, sum), P_D), _AMP_PRECISION, roundUp)
66:                     ),
67:                     Math.mul(numTokens + 1, invariant).add(
68:                         // No need to use checked arithmetic for the amp precision, the amp is guaranteed to be at least 1
69:                         Math.div(Math.mul(ampTimesTotal - _AMP_PRECISION, P_D), _AMP_PRECISION, !roundUp)
70:                     ),
71:                     roundUp
72:                 );
73: 
74:                 if (invariant > prevInvariant) {
75:                     if (invariant - prevInvariant <= 1) {
76:                         return invariant;
77:                     }
78:                 } else if (prevInvariant - invariant <= 1) {
79:                     return invariant;
80:                 }
81:             }
82:         }
83: 
84:         revert CalculationDidNotConverge();
85:     }
```

Within the `BalancerSpotPrice._calculateStableMathSpotPrice` function, the `StableMath._calculateInvariant` is computed rounding up per Line 90 below

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L90

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
```

The new Composable Pool contract uses a newer version of the StableMath library where the `StableMath._calculateInvariant` function always rounds down. Following is the StableMath.sol of one of the popular composable pools in Arbitrum that uses the new StableMath library

https://arbiscan.io/address/0xade4a71bb62bec25154cfc7e6ff49a513b491e81#code#F28#L57 (Balancer rETH-WETH Stable Pool - Arbitrum)

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

    // Always round down, to match Vyper's arithmetic (which always truncates).
    ..SNIP..
```

Thus, Notional rounds up when calculating the invariant, while Balancer's Composable Pool rounds down when calculating the invariant. This inconsistency will result in a different invariant

## Impact

High, as per past contest's risk rating - https://github.com/sherlock-audit/2022-12-notional-judging/issues/17

The invariant is used to compute the spot price to verify if the pool has been manipulated before executing certain key vault actions (e.g. reinvest rewards). If the inputted invariant is inaccurate, the spot price computed might not be accurate and might not match the actual spot price of the Balancer Pool. In the worst-case scenario, it might potentially fail to detect the pool has been manipulated, and the trade proceeds to execute against the manipulated pool, leading to a loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/math/StableMath.sol#L28

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/BalancerSpotPrice.sol#L90

## Tool used

Manual Review

## Recommendation

To avoid any discrepancy in the result, ensure that the StableMath library used by Balancer's Composable Pool and Notional's leverage vault are aligned, and the implementation of the StableMath functions is the same between them.