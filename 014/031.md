Scrawny Velvet Goose

medium

# ` params.oracleSlippagePercentOrLimit` for static trades is not checked.

## Summary
Static Trades might be settled with a large slippage causing a loss of assets as the oracleSlippagePercentOrLimit limit is not bounded, and can exceed the `Constants.SLIPPAGE_LIMIT_PRECISION` threshold.
## Vulnerability Detail
static trades allow any ` params.oracleSlippagePercentOrLimit` without check mating it, on the call to `_executeTradeWithStaticSlippage()`
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L152C1-L176C6
```solidity
    function _executeTradeWithStaticSlippage(
        ITradingModule tradingModule,
        TradeParams memory params,
        address sellToken,
        address buyToken,
        uint256 amount
    ) internal returns (uint256 amountSold, uint256 amountBought) {
        require(
            params.tradeType == TradeType.EXACT_IN_SINGLE ||
            params.tradeType == TradeType.EXACT_IN_BATCH
        );

        Trade memory trade = Trade(
            params.tradeType,
            sellToken,
            buyToken,
            amount,
            params.oracleSlippagePercentOrLimit,
            block.timestamp, // deadline
            params.exchangeData
        );

        // Execute trade using the absolute slippage limit set by `oracleSlippagePercentOrLimit`
        (amountSold, amountBought) = trade._executeTrade(params.dexId, tradingModule);
    }
```
this function is used during reward reinvestment trades.
## Impact
Trade can settle with a high slippage.
## Code Snippet
- https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L152C1-L176C6
## Tool used

Manual Review

## Recommendation
Consider restricting the slippage limit when executing static trades.