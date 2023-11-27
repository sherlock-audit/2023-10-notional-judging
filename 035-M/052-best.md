Huge Cinnamon Dalmatian

medium

# Emergency withdraw might not be enough if the underlying pool is a nested pool

## Summary
Emergency withdraw is can be called in various cases. The protocol team states that it can be called when a balancer pool enables "enableRecoveryMode" and maybe it can be called when the pool has an exploit and it needs to be unwinded immediately. However, if the actual pool is a nested pool that has an another pool lp as its underlying and that pool has the exploit or "enableRecoveryMode" toggle on then the emergency withdraw will not be able to exit the position properly since the actual pool that has the emergency issue is the nested pool.
## Vulnerability Detail
Assume that there is a CPS-wstETH-rETH, WETH weighted pool where as this weighted pool has only 2 tokens which is the weth token and the cps-wsteth-reth bpt token. If the emergency withdraw is called the vault will receive those two tokens. However, if the actual problem (maybe exploit or recovery mode toggle on) happens in the underlying pool then the emergency withdraw will not be able to cover the emergency situation. 

## Impact
It is likely to have these type of pools and it might make sense to deposit them. However, the emergency withdraw should be properly seated for these type pools to maintain the emergency functionality. Since the protocol team states that any pool that has a notional finance assets is listed is eligible for single sided strategy, and in such pools the emergency withdraw can be broken I will label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L480-L496
## Tool used

Manual Review

## Recommendation
If the pool is nested make the emergency exit also exits the underlying pool if the emergency role wants to. That way the emergency functionality can work properly for nested pools. 