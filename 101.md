Original Candy Mallard

medium

# `TokenUtils.getDecimals()` is not supporting tokens with more than 18 decimals

## Summary

`TokenUtils.getDecimals()` is not supporting tokens with decimals bigger than 18. Whenever this function gets called with a token that has more than 18 decimals, there will be a revert triggered. Thus, certain vaults can't be created with tokens that have more than 18 decimals.

## Vulnerability Detail

Whenever `TokenUtils.getDecimals()` is used with a token that has more than 18 decimals a revert will be triggered on line 14 in TokenUtils.sol.

According to the [contest page](https://audits.sherlock.xyz/contests/119) any ERC20 tokens should be supported.

> Which ERC20 tokens do you expect will interact with the smart contracts? <br>
> any

`TokenUtils.getDecimals()` is used by:

- `CrossCurrencyVault.initialize()` to set up `LEND_DECIMALS` and `BORROW_DECIMALS` (line 86-87 CrossCurrencyVault.sol) for the lend and borrow tokens.
- `BalancerComposableAuraVault` and `BalancerWeightedAuraVault` constructors to set up the decimals for the tokens (line 98-102 BalancerPoolMixin.sol)


## Impact

The Notional protocol is not compatible with tokens that have more than 18 decimals. Vaults like the `CrossCurrencyVault`, `BalancerComposableAuraVault`, `BalancerWeightedAuraVault` don't support tokens with more than 18 decimals as shown in the details above.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/utils/TokenUtils.sol#L11-L14

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/BalancerPoolMixin.sol#L98-L102

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyVault.sol#L86-L87

## Tool used

Manual Review

## Recommendation

Since any ERC20 tokens are supposed to be supported, consider supporting tokens with more than 18 decimals by adjusting the `TokenUtils.getDecimals()` function to not revert when a token has more than 18 decimals.