Shallow Peanut Elk

medium

# Hardcode Chain ID

## Summary

Leverage vault will not be able to deploy on Ethereum and Optimism due to hardcoded Chain ID.

## Vulnerability Detail

Per the contest's README, the contracts are intended to be deployed on Optimism sidechain and Ethereum Mainnet If a contract or protocol cannot be deployed on any of the mentioned chains in the README, it will be considered a valid issue in the context of this audit.

https://github.com/sherlock-audit/2023-10-notional-xiaoming9090#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed

> **Q: On what chains are the smart contracts going to be deployed?**
>
> Arbitrum, Mainnet, Optimism

However, with the current implementation based on the in-scope codebase, the contracts will not be able to deploy due to the hardcoded chain ID of 42161 (Arbitrum) at Line 59. As a result, the deployment of existing in-scope contracts will revert and fail.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/BaseStrategyVault.sol#L59

```solidity
File: BaseStrategyVault.sol
18: abstract contract BaseStrategyVault is Initializable, IStrategyVault, AccessControlUpgradeable {
..SNIP..
57:     constructor(NotionalProxy notional_, ITradingModule tradingModule_) initializer {
58:         // Make sure we are using the correct Deployments lib
59:         uint256 chainId = 42161;
60:         //assembly { chainId := chainid() }
61:         require(Deployments.CHAIN_ID == chainId);
62: 
63:         NOTIONAL = notional_;
64:         TRADING_MODULE = tradingModule_;
65:     }
```

## Impact

Leverage Vault will not be able to deploy on Ethereum and Optimism. In addition, if the affected vaults cannot be used, it leads to a loss of revenue for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/BaseStrategyVault.sol#L59

## Tool used

Manual Review

## Recommendation

Update the codebase to work with Optimism sidechain and Ethereum Mainnet