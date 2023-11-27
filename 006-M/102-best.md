Urban Tweed Pony

medium

# Approve to zero not used for lendunderlyingtokens, will affect tokens like USDT

## Summary

If lendUnderlyingTokens is USDT, then `_depositFromNotional()` will not work.

## Vulnerability Detail

The vault borow in one currency and trades it into a different currency. Some ERC20 tokens like USDT require resetting the approval to 0 first before being able to reset it to another value.  If the lend token is USDT, then must approve to zero first.

```solidity
            if (isETH) {
                WETH.deposit{value: lendUnderlyingTokens}();
                IERC20(address(WETH)).approve(address(wfCash), lendUnderlyingTokens);
            } else {
>               lendToken.approve(address(wfCash), lendUnderlyingTokens); 
            }
            vaultShares = wfCash.deposit(lendUnderlyingTokens, address(this));
        }
```


## Impact

Unsafe ERC20 approve that do not handle non-standard erc20 behavior. 1.Some token contracts do not return any value. 2.Some token contracts revert the transaction when the allowance is not zero.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/CrossCurrencyVault.sol

## Tool used

Manual Review

## Recommendation

Set the allowance to zero first before increasing the allowance, like how its done in TokenUtils `checkApprove()`

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/utils/TokenUtils.sol#L24
