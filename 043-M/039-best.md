Melodic Punch Mandrill

high

# `_checkReentrancyContext` should be called in more instances than already is, to protect against read-only reentrancy

## Summary
`_checkReentrancyContext` used in `BalancerComposableAuraVault.sol` and `BalancerWeightedAuraVault.sol` is used to protect against reentrancy, as per balancer recommendation, but it is called only in one instance in `deleverageAccount` function, which is not enough.
## Vulnerability Detail
As per balancer recommendation, it is important to protect against the reentrancy vulnerability found in multiple balancer pools, explained in details here https://forum.balancer.fi/t/reentrancy-vulnerability-scope-expanded/4345
The only case where `_checkReentrancyContext` is used right not is only in the `deleverageAccount` function, as can be seen here 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/BaseStrategyVault.sol#L217-L229
but there is also another very important case, that the `_checkReentrancyContext` should be called, and that is every time when `getActualSupply` is called. As can be seen in the NatSpec of `getActualSupply` of a random composable stable pool 
https://etherscan.io/address/0x42ed016f826165c2e5976fe5bc3df540c5ad0af7#code#F3#L1104
it is specified that it is very important for the reentrancy check to be called every time before `getActualSupply` is called, for this function to be safe to call, but this is not done in any of the contracts provided, as can be seen here 
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L108-L110
This could lead to reentrancy vulnerability, explained on the balancer forum, being exploited and that would hurt the protocol and the users.
## Impact
Impact is a high one, since the damage could be very high.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerComposableAuraVault.sol#L108-L110
## Tool used

Manual Review

## Recommendation
Call `_checkReentrancyContext` before calling `getActualSupply`, so the function call would be protected from reentrancy.