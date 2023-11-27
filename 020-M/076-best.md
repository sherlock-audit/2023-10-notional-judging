Shallow Peanut Elk

medium

# BPT LP Token could be sold off during re-investment

## Summary

BPT LP Token could be sold off during the re-investment process. BPT LP Tokens must not be sold to external DEXs under any circumstance because:

- They are used to redeem the underlying assets from the pool when someone exits the vault
- The BPTs represent the total value of the vault

## Vulnerability Detail

Within the `ConvexStakingMixin._isInvalidRewardToken` function, the implementation ensures that the LP Token (`CURVE_POOL_TOKEN`) is not intentionally or accidentally sold during the reinvestment process.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L64

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

However, the same control was not implemented for the Balancer/Aura code. As a result, it is possible for LP Token (BPT) to be sold during reinvestment. Note that for non-composable Balancer pools, the pool tokens does not consists of the BPT token. Thus, it needs to be explicitly defined within the `_isInvalidRewardToken` function.

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

LP tokens (BPT) might be accidentally or maliciously sold off by the bots during the re-investment process. BPT LP Tokens must not be sold to external DEXs under any circumstance because:

- They are used to redeem the underlying assets from the pool when someone exits the vault
- The BPTs represent the total value of the vault

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/AuraStakingMixin.sol#L38

## Tool used

Manual Review

## Recommendation

Ensure that the LP tokens cannot be sold off during re-investment.

```diff
function _isInvalidRewardToken(address token) internal override view returns (bool) {
    return (
        token == TOKEN_1 ||
        token == TOKEN_2 ||
        token == TOKEN_3 ||
        token == TOKEN_4 ||
        token == TOKEN_5 ||
+       token == BALANCER_POOL_TOKEN ||
        token == address(AURA_BOOSTER) ||
        token == address(AURA_REWARD_POOL) ||
        token == address(Deployments.WETH)
    );
}
```