Melodic Punch Mandrill

high

# `depositFromNotional` function is payable, which means that it should accept Ether, but in reality will revert 100% when msg.value > 0

## Summary
The function `depositFromNotional` used in `BaseStrategyVault.sol` is payable, which means that it should accept Ether, but in reality it will revert every time when msg.value is > than 0 in any of existing strategy.
## Vulnerability Detail
`depositFromNotional` is a function used in `BaseStrategyVault.sol` for every strategy, to deposit from notional to a specific strategy. As you can see this function has the `payable` keyword
 https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/BaseStrategyVault.sol#L166-L173
which means that it is expected to be used along with msg.value being > than 0. This function would call `_depositFromNotional` which is different on any strategy used, but let's take the most simple case, since all of them will be the same in the end, the case of `CrossCurrencyVault.sol`. In `CrossCurrencyVault.sol` , `_depositFromNotional`  would later call `_executeTrade`
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyVault.sol#L184 
which would use the `TradeHandler` library as can be seen here 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/BaseStrategyVault.sol#L124
If we look into the `TradeHandler` library, `_executeTrade` would delegatecall into the implementation of `TradingModule.sol` to `executeTrade` function, as can be seen here 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/trading/TradeHandler.sol#L41-L42
If we look into the `executeTrade` function in `TradingModule.sol` we can see that this function does not have the payable, keyword, which mean that it will not accept msg.value >  than 0 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/trading/TradingModule.sol#L169-L193
The big and important thing to know about delegate call is that, msg.sender and msg.value will always be kept when you delegate call, so the problem that arise here is the fact that , the calls made to `depositFromNotional`  with msg.value > 0 , would always revert when it gets to this delegatecall, in every strategy, since the function that it delegates to , doesn't have the `payable` keyword, and since msg.value is always kept trough the delegate calls, the call would just revert. This would be the case for all the strategies since they all uses `_executeTrade` or `_executeTradeWithDynamicSlippage` in a way or another, so every payable function that would use any of these 2 functions from the `TradeHandler.sol` library would revert all the time, if msg.value is > 0.
## Impact
Impact is a high one, since strategy using pools that need to interact with Ether would be useless and the whole functionality would be broken.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/trading/TradeHandler.sol#L18-L45
## Tool used

Manual Review

## Recommendation
There are multiple solution to this, the easiest one is to make the function `executeTrade` in `TradingModule.sol` payable, so the delegate call would not revert when msg.value is greater than 0, or if you don't intend to use ether with `depositFromNotional`, remove the `payable` keyword. It is important to take special care in those delegate calls since they are used multiple times in the codebase, and could mess up functionalities when msg.value is intended to be used.