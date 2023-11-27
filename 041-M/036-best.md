Melodic Punch Mandrill

high

# `BalancerWeightedAuraVault.sol` wrongly assumes that all of the weighted pools uses `totalSupply`

## Summary
`BalancerWeightedAuraVault.sol` which is used specifically for weighted balancer pools wrongly uses `totalSupply` all the time to get the total supply of the pool, but this would not be true for newer weighted pools.
## Vulnerability Detail
Balancer pools have different methods to get their total supply of minted LP tokens, which is also specified in the docs here 
https://docs.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html#getting-bpt-supply .
The docs specifies the fact that `totalSupply` is only used for older stable and weighted pools, and should not be used without checking it first, since the newer pools have pre-minted BPT and `getActualSupply` should be used in that case. Most of the time, the assumption would be that only the new composable stable pools uses the `getActualSupply`, but that is not the case, since even the newer weighted pools have and uses the `getActualSupply`. To give you few examples of newer weighted pools that uses `getActualSupply` 
https://etherscan.io/address/0x9f9d900462492d4c21e9523ca95a7cd86142f298
https://etherscan.io/address/0x3ff3a210e57cfe679d9ad1e9ba6453a716c56a2e
https://etherscan.io/address/0xcf7b51ce5755513d4be016b0e28d6edeffa1d52a
the last one also being on Aura finance. Because of that, the interaction with newer weighted pools and also the future weighted pools would not be accurate since the wrong total supply is used and calculations like `_mintVaultShares` and `_calculateLPTokenValue` would both be inaccurate, which would hurt the protocol and the users.
## Impact
Impact is a high one since the calculation of shares and the value of the LP tokens is very important for the protocol, and any miscalculations could hurt the protocol and the users a lot.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/BalancerWeightedAuraVault.sol#L1-L94
## Tool used

Manual Review

## Recommendation
Since the protocol would want to interact with multiple weighted pools, try to check first if the weighted pool you are interacting with is a newer one and uses the `getActualSupply` or if it is an older one and uses `totalSupply`, in that way the protocol could interact with multiple pools in the long run.