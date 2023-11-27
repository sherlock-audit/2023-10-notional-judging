Rhythmic Denim Elephant

medium

# Hardcoding `block.timestamp` should not be used as deadline

## Summary
Advanced protocols like Automated Market Makers (AMMs) can allow users to specify a deadline parameter that enforces a time limit by which the transaction must be executed. Without a deadline parameter, the transaction may sit in the mempool and be executed at a much later time potentially resulting in a worse price for the user.


## Vulnerability Detail

The function _executeTradeWithStaticSlippage() shown below is passing block.timestamp to a pool, which means that whenever the miner decides to include the txn in a block, it will be valid at that time, since block.timestamp will be the current timestamp.

```solidity
File: StrategyUtils.sol

152: function _executeTradeWithStaticSlippage(
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
            block.timestamp, // deadline            // @audit - shouldn't be set to block timestamp
            params.exchangeData
        );
```
## Impact
A miner can also just hold it until maximum slippage is incurred.
Protocols shouldn't set the deadline to `block.timestamp` as a validator can hold the transaction and the block it is eventually put into will be block.timestamp, so this offers no protection.

## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyVault.sol#L177
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyVault.sol#L248
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L140
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L170

## Tool used
Manual Review

## Recommendation
Add deadline arguments to all functions that interact with AMMs, and pass it along to AMM calls.

