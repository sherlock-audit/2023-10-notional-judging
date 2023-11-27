Future Nylon Bear

medium

# No check for active L2 Sequencer

## Summary
Using Chainlink in L2 chains such as Arbitrum requires to check if the sequencer is down to avoid prices from looking like they are fresh although they are not according to their [recommendation](https://docs.chain.link/data-feeds/l2-sequencer-feeds#arbitrum)

## Vulnerability Detail
The `SingleSidedLPVaultBase` and `CrossCurrencyVault` contracts make the `getOraclePrice` external call to the `TradingModule` contract. However, the `getOraclePrice` in the `TradingModule` makes no check to see if the sequencer is down.

## Impact
If the sequencer goes down, the protocol will allow users to continue to operate at the previous (stale) rates and this can be leveraged by malicious actors to gain unfair advantage.

## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L323

```solidity
    function _getOraclePairPrice(address base, address quote) internal view returns (uint256) {
        (int256 rate, int256 precision) = TRADING_MODULE.getOraclePrice(base, quote);
        require(rate > 0);
        require(precision > 0);
        return uint256(rate) * POOL_PRECISION() / uint256(precision);
    }
```

https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/CrossCurrencyVault.sol#L131

```solidity
        (int256 rate, int256 rateDecimals) = TRADING_MODULE.getOraclePrice(
```

https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/trading/TradingModule.sol#L71C1-L77C6

```solidity
    function getOraclePrice(address baseToken, address quoteToken)
        public
        view
        override
        returns (int256 answer, int256 decimals)
    {
        PriceOracle memory baseOracle = priceOracles[baseToken];
        PriceOracle memory quoteOracle = priceOracles[quoteToken];

        int256 baseDecimals = int256(10**baseOracle.rateDecimals);
        int256 quoteDecimals = int256(10**quoteOracle.rateDecimals);

        (/* */, int256 basePrice, /* */, uint256 bpUpdatedAt, /* */) = baseOracle.oracle.latestRoundData();
        require(block.timestamp - bpUpdatedAt <= maxOracleFreshnessInSeconds);
        require(basePrice > 0); /// @dev: Chainlink Rate Error

        (/* */, int256 quotePrice, /* */, uint256 qpUpdatedAt, /* */) = quoteOracle.oracle.latestRoundData();
        require(block.timestamp - qpUpdatedAt <= maxOracleFreshnessInSeconds);
        require(quotePrice > 0); /// @dev: Chainlink Rate Error

        answer =
            (basePrice * quoteDecimals * RATE_DECIMALS) /
            (quotePrice * baseDecimals);
        decimals = RATE_DECIMALS;
    }
```
## Tool used

Manual Review

## Recommendation

It is recommended to follow the Chailink [example code](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code)