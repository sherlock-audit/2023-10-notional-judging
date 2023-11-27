Energetic Tortilla Okapi

medium

# `SingleSidedLPVaultBase.emergencyExit` can be reverted by a front running withdrawal under certain conditions

## Summary
All the leveraged vaults share the same `emergencyExit` function, inherited from their ancestor `SingleSidedLPVaultBase`.

When `SingleSidedLPVaultBase.emergencyExit` is called with the parameter `claimToExit` being the number of claims the vault holds, then a front running withdrawal transaction will cause the emergency exit to revert.


## Vulnerability Detail
When performing an emergency exit, the used parameter `claimToExit` is the amount of claims of the leveraged vault to unstake from the reward pool and exit from the LP pool.

[SingleSidedLPVaultBase.emergencyExit](https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L480-L496)
```solidity
    function emergencyExit(
        uint256 claimToExit, bytes calldata /* data */
    ) external override onlyRole(EMERGENCY_EXIT_ROLE) {
        StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
        if (claimToExit == 0) claimToExit = state.totalPoolClaim;

        // By setting min amounts to zero, we will accept whatever tokens come from the pool
        // in a proportional exit. Front running will not have an effect since no trading will
        // occur during a proportional exit.
        _unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), true);

        state.totalPoolClaim = state.totalPoolClaim - claimToExit;
        state.setStrategyVaultState();

        emit EmergencyExit(claimToExit);
        _lockVault();
    }
```

When `claimToExit` is greater than the claims the vault holds, then the transaction reverts during `_unstakeAndExitPool` call with `"ERC20: burn amount exceeds balance"` from the pool.

### Steps
Let us assume one user: Alice, who holds 5 shares.

1. Emergency Exit tx created with `claimToExit` == `state.totalPoolClaim`
2. Withdrawal `one share` tx created 
3. Withdrawal tx is processed before Emergency Exit tx (front running)
4. Vault holds one less share, and the corresponding amount less claims. 
5. Emergency Exit tx is processed but reverts on `_unstakeAndExitPool`, as `claimToExit` is now greater than the vault's held claims.

### Alternative cause
To exit all funds, it most likely zero will be used as `claimToExit`, however is still a case where `state.totalPoolClaim` can be greater than the held claims, meaning you would legitimately pass in a non-zero value for `claimToExit`.

Accounting problems with either the reward or LP pools (e.g. from an exploit), would result in the claims accounted in the leveraged vaults being no longer in sync with those held by the pools, meaning `claimToExit` would be the value held by the pools, not `state.totalPoolClaim`.



### Proof of concept
A test case demonstrating `claimToExit` as the reward pool balance for the vault, using the `Curve2TokenConvexVault.sol`.

Place the following Solidity into the contract `Test_FRAX` in `FRAX_USDC_e.t.sol` and run with `forge test --match-contract Test_FRAX --match-test test_frontRunning_prevents_emergencyExit`
```solidity
    function test_frontRunning_prevents_emergencyExit() public {
        // Deal to 'account' and deposit into the vault
        address account = makeAddr("account");
        uint256 maturity = maturities[0];
        enterVaultBypass(account, maxDeposit, maturity, getDepositParams(0, 0));

        // Grant 'exit' the emergency exit role
        address exit = makeAddr("exit");
        grantEmergencyExitRole(exit);

        // Vault holdings of the reward pool
        uint256 rewardPoolClaim = rewardPool.balanceOf(address(vault));
        assertGt(0, rewardPoolClaim);

        // Front running tx - withdraw; single share, no minimum price
        exitVaultBypass(
            account,
            1,
            maturity,
            getRedeemParams(0, maturity)
        );

        // Emergency exit tx
        vm.prank(exit);
        v().emergencyExit(rewardPoolClaim, "");
        // @audit - emergency exit reverts with "Reason: ERC20: burn amount exceeds balance"
        // Cause being withdrawing more than balance from the Convex pool
        // 0xF2afB340D1B50108Bd32212e867946B5B8044c23::withdraw(988296665078543581 [9.882e17], false) [delegatecall]
    }

    function grantEmergencyExitRole(address exit) private {
        bytes32 role = v().EMERGENCY_EXIT_ROLE();
        vm.prank(NOTIONAL.owner());
        v().grantRole(role, exit);
    }
```


## Impact
Excerpt from the Discord channel [notional-update-4-nov-18](https://discord.com/channels/812037309376495636/1175450365395751023/1175781082336067655)

> Jeff Wu | Notional — 11/19/2023 10:54 PM
>
> Emergency exits are only intended to be executed during a security incident, not possible to really have a "verification" of such a thing on chain. 
> I'd expect whoever is holding this role to be working in the best interests of protecting user funds. 
> If they would be unable to fulfill that role then yes, I would say that is valid bug. 

The emergency exit function is intended to withdraw funds from the both the reward and LP pools in case of a security incident, an example could be either the LP pool or reward pool being attacked / exploited.

When the actor attempts to protect user funds by calling the emergency exit, they would believe there is a real risk for material loss of funds.
 i.e. emergency exit will not be called frivolously, but only in serious circumstances.

A situation where emergency exit is prevented from being successfully executing, leaves the user funds exposed risk of loss, 
which depending on the nature of the incident, could result in serious downside.

The cost of executing the front running attack is a single fraction of a vault share and the elevated gas price, 
and the attack is repeatable (keep funds at risk as long as necessary).


## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L489-L489


## Tool used
Manual Review, Forge tst


## Recommendation
`SingleSidedLPVaultBase.emergencyExit()` is only callable by a restricted role (who we may assume is acting in the best interest of the protocol).
When the input `claimToExit` is greater than the current claim the vault holds (`state.totalPoolClaim`), 
reducing the amount being claimed to the full amount of claims the vault holds will fulfill the intent of caller,
as best can be achieved in an emergency situation.

Update in [SingleSidedLPVaultBase.emergencyExit](https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L480-L496)
```diff
    function emergencyExit(
        uint256 claimToExit, bytes calldata /* data */
    ) external override onlyRole(EMERGENCY_EXIT_ROLE) {
        StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
-        if (claimToExit == 0) claimToExit = state.totalPoolClaim;
+        if (claimToExit == 0 || claimToExit > state.totalPoolClaim) claimToExit = state.totalPoolClaim;

        // By setting min amounts to zero, we will accept whatever tokens come from the pool
        // in a proportional exit. Front running will not have an effect since no trading will
        // occur during a proportional exit.
        _unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), true);

        state.totalPoolClaim = state.totalPoolClaim - claimToExit;
        state.setStrategyVaultState();

        emit EmergencyExit(claimToExit);
        _lockVault();
    }
```
