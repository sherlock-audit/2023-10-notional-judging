Huge Cinnamon Dalmatian

medium

# Emergency action is insufficient in some cases

## Summary
Emergency withdraw function is called when there is a potential case where the funds needs to be unwinded from the underlying pool. As the protocol team states in discord channel "Emergency exits are only intended to be executed during a security incident" so we can expect that an exploit in the pool can trigger an emergency withdraw call if the funds can be saved. The problem is the afterwards, if the pool is exploited then there is no point to deposit back to pool which in order for vaults to act normally the vault needs to be restored via restoreVault call which that function will try to re-deposit all the underlying tokens back to the pool. However, in such scenario doing this is unnecessary since pool is dead. All the underlying tokens can be sold to primary token and users can redeem primary token directly would be a better approach. 
## Vulnerability Detail
Assume the underlying curve pool of the vaults strategy has exploited and funds are slowly getting drained. Emergency role holder acted quickly and saved some of the funds from the pool and withdrew all the assets back to vault and vault now holds the underlying tokens including the primary token. Since the pool is exploited there is no point to deposit back to the pool. However, restoreVault, which is the only option for users to redeem their funds tries to re-deposit to the pool which is not ideal as we can see here:
```solidity
for (uint256 i; i < tokens.length; i++) {
            if (address(tokens[i]) == address(POOL_TOKEN())) continue;
            amounts[i] = TokenUtils.tokenBalance(address(tokens[i]));
        }

        // No trades are specified so this joins proportionally using the
        // amounts specified.
        uint256 poolTokens = _joinPoolAndStake(amounts, minPoolClaim);

        state.totalPoolClaim = state.totalPoolClaim + poolTokens;
        state.setStrategyVaultState();
```

However, as we said, if the pool is not functional there is no point of redepositing to pool and risk losing them back again. In such case users should be able to redeem their vault tokens for primary token directly. 

## Impact
We can say that a new upgrade on vault contract can take place and funds can be returned to the users as I described above. However, the time that requires to built a new contract and deploy it carefully with audits can take some time. Meanwhile since vault borrows from Notional this time interval can lead to late liquidations or even worse, bad debt even though the emergency withdraw called in correct time and loss of funds are not the case. Considering these, I think emergency withdraw should act quick if it successfully pulls out the funds from the pool hence, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L480-L525
## Tool used

Manual Review

## Recommendation
Make a flag that owner can set. If it is set owner can sell the underlyings to primary and users can claim their primary tokens pro-rata. If it's not setted then it means the pool is fine and the problem is something else and it will be over shortly. 