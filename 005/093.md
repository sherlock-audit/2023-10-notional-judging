Loud Grey Wolverine

medium

# The transaction of the AuraStakingMixin#`_initialApproveTokens()` may be reverted due to approving a pair token with only `type(uint256).max`

## Summary
**$COMP** and **$UNI** would be reverted if these tokens is approved with an amount, which is larger than `uint96`.
However, within the AuraStakingMixin#`_initialApproveTokens()`, each pair token would **only** be approved with `type(uint256).max`.

This lead to reverting the transaction of the AuraStakingMixin#`_initialApproveTokens()` if a pair token of Aura Pool (**$COMP Pool**) would be **$COMP**.

**$COMP** is an existing risk because the [**$COMP Pool** (50COMP-50wstETH Pool)](https://app.aura.finance/#/1/pool/90) has existed as an Aura Pool. 
If the **$UNI Pool** would be listed on Aura Pool in the future, **$UNI** will be facing this vulnerability as well. 


**$COMP** is an existing risk because the [**$COMP Pool** (50COMP-50wstETH Pool)](https://app.aura.finance/#/1/pool/90) has existed as an Aura Pool. On the other hand, **$UNI** is a potential risk in the future.

## Vulnerability Detail
Within the AuraStakingMixin#`_initialApproveTokens()`, the `tokens[i].checkApprove()` (TokenUtils#`checkApprove()`) would be called like this:
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/AuraStakingMixin.sol#L55
```solidity
    /// @notice Called once on initialization to set token approvals
    function _initialApproveTokens() internal override {
        (IERC20[] memory tokens, /* */) = TOKENS();
        for (uint256 i; i < tokens.length; i++) {
            tokens[i].checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max); ///<----- @audit
        }

        // Approve Aura to transfer pool tokens for staking
        POOL_TOKEN().checkApprove(address(AURA_BOOSTER), type(uint256).max);
    }
```

Within the TokenUtils#`checkApprove()`, the `amount` of `token` would be approved like this:
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/utils/TokenUtils.sol#L30
```solidity
    function checkApprove(IERC20 token, address spender, uint256 amount) internal {
        if (address(token) == address(0)) return;

        IEIP20NonStandard(address(token)).approve(spender, 0);
        _checkReturnCode();
        if (amount > 0) {
            IEIP20NonStandard(address(token)).approve(spender, amount); ///<------------ @audit
            _checkReturnCode();
        }
    }
```

According to the ["Revert on Large Approvals & Transfers"](https://github.com/d-xo/weird-erc20#revert-on-large-approvals--transfers), **$UNI** and **$COMP** would be reverted if these tokens is approved with an amount, which is larger than `uint96` like this:
> Some tokens (e.g. `UNI`, `COMP`) revert if the value passed to `approve` or `transfer` is larger than `uint96`.

According to the Aura Finance, the **$COMP Pool** (50COMP-50wstETH Pool) has existed as an Aura Pool like this:
https://app.aura.finance/#/1/pool/90

Based on above, in case of **$COMP Pool** on Aura, $COMP must be approved with less than `type(uint96).max`.
However, within the AuraStakingMixin#`_initialApproveTokens()`, each pair token would **only** be approved with `type(uint256).max` like this:
https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/AuraStakingMixin.sol#L55
```solidity
tokens[i].checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
```
This lead to reverting the transaction when the AuraStakingMixin#`_initialApproveTokens()` would be called.

More further, according to the Q&A, **`any` token** could be used like this:
> ### Q: Which ERC20 tokens do you expect will interact with the smart contracts? 
> any

This means that once **$UNI Pool** would be listed on Aura in the future, this vulnerability will also be problematic. 


## Impact
This vulnerability lead to reverting the transaction when the AuraStakingMixin#`_initialApproveTokens()` would be called if a pair token of Aura Pool would be **$COMP** (or **$UNI**).

As I mentioned above, **$COMP** is an existing risk because the [**$COMP Pool** (50COMP-50wstETH Pool)](https://app.aura.finance/#/1/pool/90) has existed as an Aura Pool. On the other hand, **$UNI** is a potential risk in the future.


## Code Snippet
- https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/AuraStakingMixin.sol#L55
- https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/utils/TokenUtils.sol#L30

## Tool used
- Manual Review


## Recommendation
Within the AuraStakingMixin#`_initialApproveTokens()`, consider approving a pair token with `type(uint96).max` if a pair token of Aura Pool (**$COMP Pool** or **$UNI Pool**) would be **$COMP** or **$UNI** like this:
```diff
    function _initialApproveTokens() internal override {
        (IERC20[] memory tokens, /* */) = TOKENS();
        for (uint256 i; i < tokens.length; i++) {
+           if (address(tokens[i]) == COMP_TOKEN_ADDRESS || address(tokens[i]) == UNI_TOKEN_ADDRESS)
+               tokens[i].checkApprove(address(Deployments.BALANCER_VAULT), type(uint96).max);
+           } else {
+               tokens[i].checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
+           } 
-           tokens[i].checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
        }

        // Approve Aura to transfer pool tokens for staking
        POOL_TOKEN().checkApprove(address(AURA_BOOSTER), type(uint256).max);
    }
```
