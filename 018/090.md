Huge Cinnamon Dalmatian

medium

# Vault can hold more LP than it supposed to which leads to inaccurate spot price

## Summary
Spot prices are directly derived from the pool and then compared with the chainlink price to assess whether manipulation is occurring. This approach is effective as long as measures are in place to prevent the vault from holding a significant number of LP tokens. The spot price calculation involves using one unit of token, and if the vault possesses the majority of the LP tokens in the pool, executing a single-sided removal might not align with either the spot price or the chainlink price. Notional recognizes this potential issue and imposes limitations on the number of LP tokens the vault can hold through deposits. However, if multiple users begin unwinding their LP positions, it could lead to Notional accumulating more LP tokens than it intends, potentially posing problems. I'll elaborate on this in the vulnerability details.
## Vulnerability Detail
Suppose the maximum threshold for Notional vault LP tokens that can be stored is 60%. This assertion is confirmed each time someone deposits into the vault, as evidenced by the following code snippet:
```solidity
// Checks that the vault does not own too large of a portion of the pool. If this is the case,
        // single sided exits may have a detrimental effect on the liquidity.
        uint256 maxPoolShare = VaultStorage.getStrategyVaultSettings().maxPoolShare;
        uint256 maxSupplyThreshold = (_totalPoolSupply() * maxPoolShare) / Constants.VAULT_PERCENT_BASIS;
        if (maxSupplyThreshold < state.totalPoolClaim)
            revert Errors.PoolShareTooHigh(state.totalPoolClaim, maxSupplyThreshold);
    }
```
his assumption holds true in general. However, let's consider a scenario where this limitation can be bypassed. Assume the total LP of the underlying pool is 100, with the vault holding 50 LP tokens, 40 of which are entitled to Alice (Alice has vault shares that worths 40 LP).

Now, imagine that over time, other LP providers (the other 50) begin to remove their liquidity balanced so that it now has a total of 60 LP tokens but the underlying tokens ratios are same, with 50 of these belonging to the vault. As a result, any further deposits will be prohibited because the pool holds the majority of the LP token supply. Despite this, since the liquidity providers removed their liquidity balanced, the spot price remains unchanged. However, if Alice were to perform a single-sided removal of her balance, using the spot price, an imbalance would occur because Alice holds 40 LP underlying shares, while the total pool share is 60 LP. This would lead to a significant disparity between the spot price and the actual single-sided removal.

Let's illustrate how this could be exploited. Suppose Bob owns 40 LP tokens in his wallet, and Alice has vault shares worth 40 LP tokens. Bob decides to remove balanced liquidity using his 40 LP tokens, thereby decreasing the total LP token supply to 60. Consequently, when calculating the LP token value, the spot price will remain unchanged due to Bob's balanced removal. However, even though the pool now holds 60 LP tokens, with 50 being vault shares and 40 belonging to Alice, if Alice were to be liquidated, resulting in the removal of 40 LP tokens from the vault in a single-sided manner, the spot price would be inaccurate. This discrepancy arises because the removal of nearly 80% of the pool's liquidity through the single-sided removal results in a worse price in terms of the primary token.

As we can see in the following snippet that as long as the liquidity are removed balanced the spot price will remain unchanged regardless of the total LP tokens supply.  
```solidity
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
```
## Impact
Vault tries to not have more LP tokens but it only does that when deposits. However, pool can have more than it should LP tokens either naturally by people removing their liquidity or a whale removing liquidity and using this in advantage in the Notional ecosystem. Since it can be used by a malicious attacker but the attacker needs considerable amount of LP's  I will label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L97-L115
## Tool used

Manual Review

## Recommendation
