Shallow Peanut Elk

high

# Unable to reinvest if the reward token equals one of the pool tokens

## Summary

If the reward token is the same as one of the pool tokens, the protocol would not be able to reinvest such a reward token. Thus leading to a loss of assets for the vault shareholders.

## Vulnerability Detail

During the reinvestment process, the `reinvestReward` function will be executed once for each reward token. The length of the `trades` listing defined in the payload must be the same as the number of tokens in the pool per Line 339 below.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L385

```solidity
File: SingleSidedLPVaultBase.sol
385:     function reinvestReward(
386:         SingleSidedRewardTradeParams[] calldata trades,
387:         uint256 minPoolClaim
388:     ) external whenNotLocked onlyRole(REWARD_REINVESTMENT_ROLE) returns (
389:         address rewardToken,
390:         uint256 amountSold,
391:         uint256 poolClaimAmount
392:     ) {
393:         // Will revert if spot prices are not in line with the oracle values
394:         _checkPriceAndCalculateValue();
395: 
396:         // Require one trade per token, if we do not want to buy any tokens at a
397:         // given index then the amount should be set to zero. This applies to pool
398:         // tokens like in the ComposableStablePool.
399:         require(trades.length == NUM_TOKENS());
400:         uint256[] memory amounts;
401:         (rewardToken, amountSold, amounts) = _executeRewardTrades(trades);
```

In addition, due to the requirement at Line 105, each element in the `trades` listing must be a token within a pool and must be ordered in sequence according to the token index of the pool.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L90

```solidity
File: StrategyUtils.sol
090:     function executeRewardTrades(
091:         IERC20[] memory tokens,
092:         SingleSidedRewardTradeParams[] calldata trades,
093:         address rewardToken,
094:         address poolToken
095:     ) external returns(uint256[] memory amounts, uint256 amountSold) {
096:         amounts = new uint256[](trades.length);
097:         for (uint256 i; i < trades.length; i++) {
098:             // All trades must sell the same token.
099:             require(trades[i].sellToken == rewardToken);
100:             // Bypass certain invalid trades
101:             if (trades[i].amount == 0) continue;
102:             if (trades[i].buyToken == poolToken) continue;
103: 
104:             // The reward trade can only purchase tokens that go into the pool
105:             require(trades[i].buyToken == address(tokens[i]));
```

Assuming the TriCRV Curve pool (crvUSD+WETH+CRV) has two reward tokens (CRV & CVX). This example is taken from a live Curve pool on Ethereum ([Reference 1](https://curve.fi/#/ethereum/pools/factory-tricrypto-4/deposit) [Reference 2](https://www.convexfinance.com/stake/ethereum/211))

The pool will consist of the following tokens:

```solidity
tokens[0] = crvUSD
tokens[1] = WETH
tokens[2] = CRV
```

Thus, if the protocol receives 3000 CVX reward tokens and it intends to sell 1000 CVX for crvUSD and 1000 CVX for WETH.

The `trades` list has to be defined as below.

```solidity
trades[0].sellToken[0] = CRV (rewardToken) | trades[0].buyToken = crvUSD | trades[0].amount = 1000
trades[1].sellToken[1] = CRV (rewardToken) | trades[1].buyToken = WETH    | trades[0].amount = 1000
trades[1].sellToken[2] = CRV (rewardToken) | trades[1].buyToken = CRV    | trades[0].amount = 0
```

The same issue also affects the Balancer pools. Thus, the example is omitted for brevity. One of the affected Balancer pools is as follows, where the reward token is also one of the pool tokens.

- WETH-AURA - [Reference 1](https://app.balancer.fi/#/ethereum/pool/0xcfca23ca9ca720b6e98e3eb9b6aa0ffc4a5c08b9000200000000000000000274) [Reference 2](https://app.aura.finance/#/1/pool/100) (Reward Tokens = [BAL, AURA])

However, the issue is that the `_isInvalidRewardToken` function within the `_executeRewardTrades` will always revert.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L413

```solidity
File: SingleSidedLPVaultBase.sol
413:     function _executeRewardTrades(SingleSidedRewardTradeParams[] calldata trades) internal returns (
414:         address rewardToken,
415:         uint256 amountSold,
416:         uint256[] memory amounts
417:     ) {
418:         // The sell token on all trades must be the same (checked inside executeRewardTrades) so
419:         // just validate here that the sellToken is a valid reward token (i.e. none of the tokens
420:         // used in the regular functioning of the vault).
421:         rewardToken = trades[0].sellToken;
422:         if (_isInvalidRewardToken(rewardToken)) revert Errors.InvalidRewardToken(rewardToken);
423:         (IERC20[] memory tokens, /* */) = TOKENS();
424:         (amounts, amountSold) = StrategyUtils.executeRewardTrades(
425:             tokens, trades, rewardToken, address(POOL_TOKEN())
426:         );
427:     }
```

The reason is that within the `_isInvalidRewardToken` function it checks if the reward token to be sold is any of the pool tokens. In this case, the condition will be evaluated to be true, and a revert will occur. As a result, the protocol would not be able to reinvest such reward tokens.

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

## Impact

The reinvestment of reward tokens is a critical component of the vault. The value per vault share increases when reward tokens are sold for the pool tokens and reinvested back into the Curve/Balancer pool to obtain more LP tokens. If this feature does not work as intended, it will lead to a loss of assets for the vault shareholders.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L385

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/AuraStakingMixin.sol#L38

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/ConvexStakingMixin.sol#L60

## Tool used

Manual Review

## Recommendation

Consider tracking the number of pool tokens received during an emergency exit, and segregate these tokens with the reward tokens. For instance, the vault has 3000 CVX, 1000 of them are received during the emergency exit, while the rest are reward tokens emitted from Convex/Aura. In this case, the protocol can sell all CVX on the vault except for the 1000 CVX reserved.