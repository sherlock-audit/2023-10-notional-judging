Shallow Peanut Elk

medium

# Leverage vault will break in Optimism sidechain

## Summary

Leverage vault for Curve pool will not work on Optimism sidechain.

## Vulnerability Detail

Per the contest's README, the contracts are intended to be deployed on Optimism sidechain. If a contract or protocol cannot be deployed on any of the mentioned chains in the README, it will be considered a valid issue in the context of this audit.

https://github.com/sherlock-audit/2023-10-notional-xiaoming9090#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed

> **Q: On what chains are the smart contracts going to be deployed?**
>
> Arbitrum, Mainnet, Optimism

It was observed that the current implementation in the in-scope codebase will not work on Optimism sidechain.

Following are a few instances of the incompatibilities observed:

**Instance 1**

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L63

```solidity
File: Curve2TokenConvexVault.sol
26:     function _joinPoolAndStake(
27:         uint256[] memory _amounts, uint256 minPoolClaim
..SNIP..
56:         // Method signatures are slightly different on mainnet and arbitrum
57:         bool success;
58:         if (Deployments.CHAIN_ID == Constants.CHAIN_ID_MAINNET) {
59:             success = IConvexBooster(CONVEX_BOOSTER).deposit(CONVEX_POOL_ID, lpTokens, true);
60:         } else if (Deployments.CHAIN_ID == Constants.CHAIN_ID_ARBITRUM) {
61:             success = IConvexBoosterArbitrum(CONVEX_BOOSTER).deposit(CONVEX_POOL_ID, lpTokens);
62:         }
63:         require(success);
​
```

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L76

```solidity
File: Curve2TokenConvexVault.sol
66:     function _unstakeAndExitPool(
67:         uint256 poolClaim, uint256[] memory _minAmounts, bool isSingleSided
..SNIP..
71:         if (Deployments.CHAIN_ID == Constants.CHAIN_ID_MAINNET) {
72:             success = IConvexRewardPool(CONVEX_REWARD_POOL).withdrawAndUnwrap(poolClaim, false);
73:         } else if (Deployments.CHAIN_ID == Constants.CHAIN_ID_ARBITRUM) {
74:             success = IConvexRewardPoolArbitrum(CONVEX_REWARD_POOL).withdraw(poolClaim, false);
75:         }
76:         require(success);
```

The `success` variable will always be false for Optimism for the above code, and result in a revert.

**Instance 2**

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L77

```solidity
File: ConvexStakingMixin.sol
71:     function _claimRewardTokens() internal override {
72:         if (Deployments.CHAIN_ID == Constants.CHAIN_ID_MAINNET) {
73:             require(IConvexRewardPool(CONVEX_REWARD_POOL).getReward(address(this), true));
74:         } else if (Deployments.CHAIN_ID == Constants.CHAIN_ID_ARBITRUM) {
75:             IConvexRewardPoolArbitrum(CONVEX_REWARD_POOL).getReward(address(this));
76:         } else {
77:             revert();
78:         }
79:     }
```

The `_claimRewardTokens()` function will always revert in Optimism

**Instance 3**

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L46

```solidity
File: ConvexStakingMixin.sol
27:     constructor(NotionalProxy notional_, ConvexVaultDeploymentParams memory params) 
28:         Curve2TokenPoolMixin(notional_, params.baseParams) {
29:         CONVEX_REWARD_POOL = params.rewardPool;
30: 
31:         address convexBooster;
32:         uint256 poolId;
33: 
34:         if (Deployments.CHAIN_ID == Constants.CHAIN_ID_MAINNET) {
35:             IConvexRewardPool rewardPool = IConvexRewardPool(CONVEX_REWARD_POOL);
36: 
37:             convexBooster = rewardPool.operator();
38:             poolId = rewardPool.pid();
39: 
40:         } else if (Deployments.CHAIN_ID == Constants.CHAIN_ID_ARBITRUM) {
41:             IConvexRewardPoolArbitrum rewardPool = IConvexRewardPoolArbitrum(CONVEX_REWARD_POOL);
42: 
43:             convexBooster = rewardPool.convexBooster();
44:             poolId = rewardPool.convexPoolId();
45:         } else {
46:             revert("Unsupported chain");
47:         }
```

The `ConvexStakingMixin` contract will fail during deployment due to revert at Line 46

## Impact

Leverage vault for Curve pool will not work on Optimism sidechain.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L63

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L76

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L77

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L46

## Tool used

Manual Review

## Recommendation

Update the existing codebase to work with Optimism sidechain.