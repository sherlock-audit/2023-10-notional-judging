Shallow Peanut Elk

high

# Leverage Vault on sidechains that support Curve V2 pools is broken

## Summary

No users will be able to deposit to the Leverage Vault on Arbitrum and Optimism that supports Curve V2 pools, leading to the core contract functionality of a vault being broken and a loss of revenue for the protocol.

## Vulnerability Detail

Following are examples of some Curve V2 pools in Arbitum:

- fETH/ETH/xETH (https://curve.fi/#/arbitrum/pools/factory-tricrypto-2/deposit)
- tricrypto (https://curve.fi/#/arbitrum/pools/tricrypto/deposit)
- eursusd (https://curve.fi/#/arbitrum/pools/eursusd/deposit)

The code from Line 64 to Line 71 is only executed if the contract resides on Ethereum. As a result, for Arbitrum and Optimism sidechains, the `IS_CURVE_V2` variable is always false.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/Curve2TokenPoolMixin.sol#L73

```solidity
File: Curve2TokenPoolMixin.sol
56:     constructor(
57:         NotionalProxy notional_,
58:         DeploymentParams memory params
59:     ) SingleSidedLPVaultBase(notional_, params.tradingModule) {
60:         CURVE_POOL = params.pool;
61: 
62:         bool isCurveV2 = false;
63:         if (Deployments.CHAIN_ID == Constants.CHAIN_ID_MAINNET) {
64:             address[10] memory handlers = 
65:                 Deployments.CURVE_META_REGISTRY.get_registry_handlers_from_pool(address(CURVE_POOL));
66: 
67:             require(
68:                 handlers[0] == Deployments.CURVE_V1_HANDLER ||
69:                 handlers[0] == Deployments.CURVE_V2_HANDLER
70:             ); // @dev unknown Curve version
71:             isCurveV2 = (handlers[0] == Deployments.CURVE_V2_HANDLER);
72:         }
73:         IS_CURVE_V2 = isCurveV2;
```

As a result, code within the `_joinPoolAndStake` function will always call the Curve V1's `add_liquidity` function that does not define the `use_eth` parameter.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L51

```solidity
File: Curve2TokenConvexVault.sol
26:     function _joinPoolAndStake(
..SNIP..
45:         // Slightly different method signatures in v1 and v2
46:         if (IS_CURVE_V2) {
47:             lpTokens = ICurve2TokenPoolV2(CURVE_POOL).add_liquidity{value: msgValue}(
48:                 amounts, minPoolClaim, 0 < msgValue // use_eth = true if msgValue > 0
49:             );
50:         } else {
51:             lpTokens = ICurve2TokenPoolV1(CURVE_POOL).add_liquidity{value: msgValue}(
52:                 amounts, minPoolClaim
53:             );
54:         }
```

If the `use_eth` parameter is not defined, it will default to `False`. As a result, the Curve pool expects the caller to transfer over the WETH to the pool and the pool will call `WETH.withdraw` to unwrap the WETH to Native ETH as shown in the code below.

However, Notional's leverage vault only works with Native ETH, and if one of the pool tokens is WETH, it will explicitly convert the address to either the `Deployments.ALT_ETH_ADDRESS` (0xEeeee) or `Deployments.ETH_ADDRESS` (address(0)) during deployment and initialization.

The implementation of the above `_joinPoolAndStake` function will forward Native ETH to the Curve Pool, while the pool expects the vault to transfer in WETH. As a result, a revert will occur since the pool did not receive the WETH it required during the unwrap process.

https://arbiscan.io/address/0xf7fed8ae0c5b78c19aadd68b700696933b0cefd9#code#L509 (Taken from Curve V2 fETH/ETH/xETH pool)

```python
def add_liquidity(
    amounts: uint256[N_COINS],
    min_mint_amount: uint256,
    use_eth: bool = False,
    receiver: address = msg.sender
) -> uint256:
    """
    @notice Adds liquidity into the pool.
    @param amounts Amounts of each coin to add.
    @param min_mint_amount Minimum amount of LP to mint.
    @param use_eth True if native token is being added to the pool.
    @param receiver Address to send the LP tokens to. Default is msg.sender
    @return uint256 Amount of LP tokens received by the `receiver
    """
..SNIP..
    # --------------------- Get prices, balances -----------------------------
..SNIP..
    # -------------------------------------- Update balances and calculate xp.
..SNIP...
    # ---------------- transferFrom token into the pool ----------------------

    for i in range(N_COINS):

        if amounts[i] > 0:

            if coins[i] == WETH20:

                self._transfer_in(
                    coins[i],
                    amounts[i],
                    0,  # <-----------------------------------
                    msg.value,  #                             | No callbacks
                    empty(address),  # <----------------------| for
                    empty(bytes32),  # <----------------------| add_liquidity.
                    msg.sender,  #                            |
                    empty(address),  # <-----------------------
                    use_eth
                )
```

```python
def _transfer_in(
..SNIP..
    use_eth: bool
):
..SNIP..
    @params use_eth True if the transfer is ETH, False otherwise.
    """

    if use_eth and _coin == WETH20:
        assert mvalue == dx  # dev: incorrect eth amount
    else:
..SNIP..
        if _coin == WETH20:
            WETH(WETH20).withdraw(dx)  # <--------- if WETH was transferred in
            #           previous step and `not use_eth`, withdraw WETH to ETH.
```

## Impact

No users will be able to deposit to the Leverage Vault on Arbitrum and Optimism that supports Curve V2 pools. The deposit function is a core function of any vault. Thus, this issue breaks the core contract functionality of a vault.

In addition, if the affected vaults cannot be used, it leads to a loss of revenue for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/curve/Curve2TokenPoolMixin.sol#L73

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L51

## Tool used

Manual Review

## Recommendation

Ensure the `IS_CURVE_V2` variable is initialized on the Arbitrum and Optimism side chains according to the Curve Pool's version.

If there is a limitation on the existing approach to determining a pool is V1 or V2 on Arbitrum and Optimsim, an alternative approach might be to use the presence of a `gamma()` function as an indicator of pool type