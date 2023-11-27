Shallow Peanut Elk

medium

# ETH can be sold during reinvestment

## Summary

The existing control to prevent ETH from being sold during reinvestment can be bypassed, allowing the bots to accidentally or maliciously sell off the non-reward assets of the vault.

## Vulnerability Detail

Multiple instances of this issue were found:

**Instance 1 - Curve's Implementation**

The `_isInvalidRewardToken` function attempts to prevent the callers from selling away ETH during reinvestment.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L60

```solidity
File: ConvexStakingMixin.sol
60:     function _isInvalidRewardToken(address token) internal override view returns (bool) {
61:         return (
62:             token == TOKEN_1 ||
63:             token == TOKEN_2 ||
64:             token == address(CURVE_POOL_TOKEN) ||
65:             token == address(CONVEX_REWARD_POOL) ||
66:             token == address(CONVEX_BOOSTER) ||
67:             token == Deployments.ALT_ETH_ADDRESS
68:         );
69:     }
```

However, the code at Line 67 above will not achieve the intended outcome as `Deployments.ALT_ETH_ADDRESS` is not a valid token address in the first place.

```solidity
address internal constant ALT_ETH_ADDRESS = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
```

When the caller is executing a trade with ETH, the address for ETH used is either `Deployments.WETH` or `Deployments.ETH_ADDRESS` (`address(0)`) as shown in the TradingUtils's source code, not the `Deployments.ALT_ETH_ADDRESS`. 

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/trading/TradingUtils.sol#L128

```solidity
File: TradingUtils.sol
128:     function _executeTrade(
129:         address target,
130:         uint256 msgValue,
131:         bytes memory params,
132:         address spender,
133:         Trade memory trade
134:     ) private {
135:         uint256 preTradeBalance;
136:  
137:         if (trade.buyToken == address(Deployments.WETH)) {
138:             preTradeBalance = address(this).balance;
139:         } else if (trade.buyToken == Deployments.ETH_ADDRESS || _needsToUnwrapExcessWETH(trade, spender)) {
140:             preTradeBalance = IERC20(address(Deployments.WETH)).balanceOf(address(this));
141:         }
```

As a result, the caller (bot) of the reinvestment function could still sell off the ETH from the vault, bypassing the requirement.

**Instance 2 - Balancer's Implementation**

When the caller is executing a trade with ETH, the address for ETH used is either `Deployments.WETH` or `Deployments.ETH_ADDRESS` (`address(0)`), as mentioned earlier. However, the `AuraStakingMixin._isInvalidRewardToken` function only blocks `Deployments.WETH` but not `Deployments.ETH`, thus allowing the caller (bot) of the reinvestment function, could still sell off the ETH from the vault, bypassing the requirement.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/AuraStakingMixin.sol#L38

```solidity
File: AuraStakingMixin.sol
38:     function _isInvalidRewardToken(address token) internal override view returns (bool) {
39:         return (
40:             token == TOKEN_1 ||
41:             token == TOKEN_2 ||
42:             token == TOKEN_3 ||
43:             token == TOKEN_4 ||
44:             token == TOKEN_5 ||
45:             token == address(AURA_BOOSTER) ||
46:             token == address(AURA_REWARD_POOL) ||
47:             token == address(Deployments.WETH)
48:         );
49:     }
```

Per the sponsor's clarification below, the contracts should protect against the bot doing unintended things (including acting maliciously) due to coding errors, which is one of the main reasons for having the `_isInvalidRewardToken` function. Thus, this issue is a valid bug in the context of this audit contest.

https://discord.com/channels/812037309376495636/1175450365395751023/1175781082336067655

![](https://user-images.githubusercontent.com/102820284/285566722-10739ec2-fc8f-43f5-b681-459494d1b6dc.png)

## Impact

The existing control to prevent ETH from being sold during reinvestment can be bypassed, allowing the bots to accidentally or maliciously sell off the non-reward assets of the vault.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L60

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/AuraStakingMixin.sol#L38

## Tool used

Manual Review

## Recommendation

To ensure that ETH cannot be sold off during reinvestment, consider the following changes:

**Curve**

```diff
File: ConvexStakingMixin.sol
function _isInvalidRewardToken(address token) internal override view returns (bool) {
    return (
        token == TOKEN_1 ||
        token == TOKEN_2 ||
        token == address(CURVE_POOL_TOKEN) ||
        token == address(CONVEX_REWARD_POOL) ||
        token == address(CONVEX_BOOSTER) ||
+       token == Deployments.ETH ||
+       token == Deployments.WETH ||
        token == Deployments.ALT_ETH_ADDRESS
    );
}
```

**Balancer**

```diff
File: AuraStakingMixin.sol
function _isInvalidRewardToken(address token) internal override view returns (bool) {
    return (
        token == TOKEN_1 ||
        token == TOKEN_2 ||
        token == TOKEN_3 ||
        token == TOKEN_4 ||
        token == TOKEN_5 ||
        token == address(AURA_BOOSTER) ||
        token == address(AURA_REWARD_POOL) ||
+		token == address(Deployments.ETH)        
        token == address(Deployments.WETH)
    );
}
```