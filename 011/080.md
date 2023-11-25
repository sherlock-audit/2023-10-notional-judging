Shallow Peanut Elk

high

# Re-enter with all tokens causing the vault to be vulnerable to donation attack

## Summary

When restoring the vault, all the tokens residing in the vault are re-entered into the pool, which results in the vault being vulnerable to donation attacks.

It is critical that the vault shares cannot be manipulated because they are used in Notional V3 when computing key parameters such as the account's health, collateral ratio, and liquidation. If these key parameters can be inflated or deflated, it might lead to loss of assets due to various issues (e.g., over-borrowing).

## Vulnerability Detail

When restoring the vault, all the tokens residing in the vault are re-entered into the pool, which results in the vault being vulnerable to donation attacks.

Assume that before the emergency exit, there are:

- 10 vault shares
- 10 BPT staked worth a total of 100 USDC and 100 USDT
- Value per share is 20 USD per vault share (200 USD value / 10 shares)

After the emergency exit (ignoring slippage for simplicity's sake), the vault will redeem 10 BPT and hold 100 USDC + 100 USDT

When the vault restoration TX is submitted, a malicious user could front-run the transaction and donate 100 USDC and 100 USDT to the vault. As such, the total number of tokens re-entering the pool will be 200 USDC + 200 USDT, which will mint 20 BPT to the vault.

After the attack, the value per share doubled to 40 USD (400 USD value / 10 shares).

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L514

```solidity
File: SingleSidedLPVaultBase.sol
501:     function restoreVault(
..SNIP..
512:         for (uint256 i; i < tokens.length; i++) {
513:             if (address(tokens[i]) == address(POOL_TOKEN())) continue;
514:             amounts[i] = TokenUtils.tokenBalance(address(tokens[i]));
515:         }
516: 
517:         // No trades are specified so this joins proportionally using the
518:         // amounts specified.
519:         uint256 poolTokens = _joinPoolAndStake(amounts, minPoolClaim);
520: 
521:         state.totalPoolClaim = state.totalPoolClaim + poolTokens;
522:         state.setStrategyVaultState();
523: 
524:         _unlockVault();
525:     }
```

## Impact

Notional explicitly guard against donation attacks in the other parts of the vault by keeping track of the tokens sent and received in the state variable to ensure that the value of the vault shares cannot be manipulated. However, the existing control is insufficient, which allows a malicious user to manipulate the value of the vault shares.

It is critical that the vault shares cannot be manipulated because they are used in Notional V3 when computing key parameters such as the account's health, collateral ratio, and liquidation. If these key parameters can be inflated or deflated, it might lead to loss of assets due to various issues (e.g., over-borrowing).

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L514

## Tool used

Manual Review

## Recommendation

Keep track of the number of tokens received during the emergency exit. During vault restoration, only re-enter with these amounts to the pool.