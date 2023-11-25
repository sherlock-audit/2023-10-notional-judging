Shallow Peanut Elk

high

# Native ETH not received when removing liquidity from Curve V2 pools

## Summary

Native ETH was not received when removing liquidity from Curve V2 pools due to the mishandling of Native ETH and WETH, leading to a loss of assets.

## Vulnerability Detail

Curve V2 pool will always wrap to WETH and send to leverage vault unless the `use_eth` is explicitly set to `True`. Otherwise, it will default to `False`. The following implementation of the `remove_liquidity_one_coin` function taken from one of the Curve V2 pools shows that unless the `use_eth` is set to `True`, the `WETH.deposit()` will be triggered to wrap the ETH, and WETH will be transferred back to the caller. The same is true for the `remove_liquidity` function, but it is omitted for brevity.

https://etherscan.io/address/0x0f3159811670c117c372428d4e69ac32325e4d0f#code

```python
@external
@nonreentrant('lock')
def remove_liquidity_one_coin(token_amount: uint256, i: uint256, min_amount: uint256,
                              use_eth: bool = False, receiver: address = msg.sender) -> uint256:
    A_gamma: uint256[2] = self._A_gamma()

    dy: uint256 = 0
    D: uint256 = 0
    p: uint256 = 0
    xp: uint256[N_COINS] = empty(uint256[N_COINS])
    future_A_gamma_time: uint256 = self.future_A_gamma_time
    dy, p, D, xp = self._calc_withdraw_one_coin(A_gamma, token_amount, i, (future_A_gamma_time > 0), True)
    assert dy >= min_amount, "Slippage"

    if block.timestamp >= future_A_gamma_time:
        self.future_A_gamma_time = 1

    self.balances[i] -= dy
    CurveToken(self.token).burnFrom(msg.sender, token_amount)

    coin: address = self.coins[i]
    if use_eth and coin == WETH20:
        raw_call(receiver, b"", value=dy)
    else:
        if coin == WETH20:
            WETH(WETH20).deposit(value=dy)
        response: Bytes[32] = raw_call(
            coin,
            _abi_encode(receiver, dy, method_id=method_id("transfer(address,uint256)")),
            max_outsize=32,
        )
        if len(response) != 0:
            assert convert(response, bool)
```

Notional's Leverage Vault only works with Native ETH. It was found that the `remove_liquidity_one_coin` and `remove_liquidity` functions are executed without explicitly setting the `use_eth` parameter to `True`. Thus, WETH instead of Native ETH will be returned during remove liquidity. As a result, these WETH will not be accounted for in the vault and result in a loss of assets.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L83C17-L83C77

```solidity
File: Curve2TokenConvexVault.sol
66:     function _unstakeAndExitPool(
..SNIP..
78:         ICurve2TokenPool pool = ICurve2TokenPool(CURVE_POOL);
79:         exitBalances = new uint256[](2);
80:         if (isSingleSided) {
81:             // Redeem single-sided
82:             exitBalances[_PRIMARY_INDEX] = pool.remove_liquidity_one_coin(
83:                 poolClaim, int8(_PRIMARY_INDEX), _minAmounts[_PRIMARY_INDEX]
84:             );
85:         } else {
86:             // Redeem proportionally, min amounts are rewritten to a fixed length array
87:             uint256[2] memory minAmounts;
88:             minAmounts[0] = _minAmounts[0];
89:             minAmounts[1] = _minAmounts[1];
90: 
91:             uint256[2] memory _exitBalances = pool.remove_liquidity(poolClaim, minAmounts);
92:             exitBalances[0] = _exitBalances[0];
93:             exitBalances[1] = _exitBalances[1];
94:         }
```

## Impact

Following are some of the impacts due to the mishandling of Native ETH and WETH during liquidity removal in Curve pools, leading to loss of assets:

1. Within the `redeemFromNotional`, if the vaults consist of ETH, the `_UNDERLYING_IS_ETH` will be set to true. In this case, the code will attempt to call `transfer` to transfer Native ETH, which will fail as Native ETH is not received and users/Notional are unable to redeem.

   ```solidity
   File: BaseStrategyVault.sol
   175:     function redeemFromNotional(
   ..SNIP..
   199:         if (_UNDERLYING_IS_ETH) {
   200:             if (transferToReceiver > 0) payable(receiver).transfer(transferToReceiver);
   201:             if (transferToNotional > 0) payable(address(NOTIONAL)).transfer(transferToNotional);
   202:         } else {
   ..SNIP..
   ```

2. WETH will be received instead of Native ETH during the emergency exit. During vault restoration, WETH is not re-entered into the pool as only Native ETH residing in the vault will be transferred to the pool. Leverage vault only works with Native ETH, and if one of the pool tokens is WETH, it will be converted to Native ETH (0x0 or 0xEeeee) during deployment/initialization. Thus, the WETH is stuck in the vault. This causes the value per share to drop significantly. ([Reference](https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L514))

   ```solidity
   File: SingleSidedLPVaultBase.sol
   480:     function emergencyExit(
   481:         uint256 claimToExit, bytes calldata /* data */
   482:     ) external override onlyRole(EMERGENCY_EXIT_ROLE) {
   483:         StrategyVaultState memory state = VaultStorage.getStrategyVaultState();
   484:         if (claimToExit == 0) claimToExit = state.totalPoolClaim;
   485: 
   486:         // By setting min amounts to zero, we will accept whatever tokens come from the pool
   487:         // in a proportional exit. Front running will not have an effect since no trading will
   488:         // occur during a proportional exit.
   489:         _unstakeAndExitPool(claimToExit, new uint256[](NUM_TOKENS()), true);
   490: 
   491:         state.totalPoolClaim = state.totalPoolClaim - claimToExit;
   492:         state.setStrategyVaultState();
   ```

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L83C17-L83C77

## Tool used

Manual Review

## Recommendation

If one of the pool tokens is ETH, consider setting `is_eth` to true when calling `remove_liquidity_one_coin` and `remove_liquidity` functions to ensure that Native ETH is sent back to the vault.