Shallow Peanut Elk

medium

# BPT could be brought during deposit trade

## Summary

BPT could be brought during deposit trade. Since the join operation of the Balancer pool does not accept BPT, the excess BPT swapped will be useless. This resulted in fewer LP tokens being minted.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L36

```solidity
File: StrategyUtils.sol
25:     /// @notice Trades the amount of primary token into other secondary tokens prior
26:     /// to entering a pool.
27:     function executeDepositTrades(
28:         IERC20[] memory tokens,
29:         uint256[] memory amounts,
30:         DepositTradeParams[] memory depositTrades,
31:         uint256 primaryIndex
32:     ) external returns (uint256[] memory) {
33:         address primaryToken = address(tokens[primaryIndex]);
34: 
35:         for (uint256 i; i < amounts.length; i++) {
36:             if (i == primaryIndex) continue;
37:             DepositTradeParams memory t = depositTrades[i];
38:             // Do not allow ZERO_EX trading in this method since we cannot validate
39:             // the arbitrary exchange data.
40:             if (DexId(t.tradeParams.dexId) == DexId.ZERO_EX) revert Errors.InvalidDexId(uint256(DexId.ZERO_EX));
41: 
42:             if (t.tradeAmount > 0) {
43:                 // Always selling the primaryToken and buying the secondary token.
44:                 (uint256 amountSold, uint256 amountBought) = _executeDynamicSlippageTradeExactIn(
45:                     Deployments.TRADING_MODULE, t.tradeParams, primaryToken, address(tokens[i]), t.tradeAmount
46:                 );
..SNIP..
```

For the composable pool, the `TOKENS()` returned will consist of the Balancer LP Token (BPT)

The validation at Line 36 ensures that the primary token cannot be sold in exchange for the primary token. However, it does not ensure that the primary token cannot be sold in exchange for the Balancer LP Token (BPT). 

## Impact

BPT could be brought during deposit trade. Since the join operation of the Balancer pool does not accept BPT, the excess BPT swapped will be useless. This resulted in fewer LP tokens being minted.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L36

## Tool used

Manual Review

## Recommendation

Consider adding an additional validation to ensure that the buy token cannot be BPT.

```diff
function executeDepositTrades(
    IERC20[] memory tokens,
    uint256[] memory amounts,
    DepositTradeParams[] memory depositTrades,
    uint256 primaryIndex
) external returns (uint256[] memory) {
    address primaryToken = address(tokens[primaryIndex]);

    for (uint256 i; i < amounts.length; i++) {
        if (i == primaryIndex) continue;
+       if (i == BPT_INDEX) continue;
```