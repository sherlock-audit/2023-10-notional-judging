Melodic Punch Mandrill

medium

# Most of the composable stable pools uses wstETH on the mainnet, which is would not work with the current codebase because of `_getOraclePairPrice`

## Summary
Most of the composable stable pool on Balancer uses wstETH as one of the tokens, but that would not be compatible with `BalancerComposableAuraVault.sol`  because there is no mainnet oracle compatible with wstETH, so the function `_getOraclePairPrice` used in `_calculateLPTokenValue` would not be possible to be used.
## Vulnerability Detail
As you can see on the balancer composable stable pools page for mainnet 
https://app.balancer.fi/#/ethereum
most of the pools, and especially the ones with the biggest values, uses wstETH as one of their assets. The problem with the current codebase is that wstETH would not be compatible with the current `_getOraclePairPrice` function. As you can see `_checkPriceAndCalculateValue` calls `_calculateLPTokenValue` 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L104
which calls `_getOraclePairPrice`
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L351
if the assets provided is not the primary index one. Now if we look at the `_getOraclePairPrice` function, it calls `getOraclePrice` on the `TRADING_MODULE` contract 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L323
which uses the chainlink oracle to get the latest round data for those 2 assets provided in the `_getOraclePairPrice` function 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/trading/TradingModule.sol#L237-L261
but since wstETH is not supported on any mainnet oracle, the whole function could not be used. To be more specifically, there will be issues when anytime when `_checkPriceAndCalculateValue`, which is on `reinvestReward` , `getExchangeRate` 
 and `convertStrategyToUnderlying` functions in `SingleSidedLPVaultBase.sol`.
## Impact
Impact is a medium one since reinvesting and other functionalities would not be possible, in those cases.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L322-L327
## Tool used

Manual Review

## Recommendation
Use another way of checking the price of wstETH, since avoiding interacting with it would make using most of the composable pool impossible.