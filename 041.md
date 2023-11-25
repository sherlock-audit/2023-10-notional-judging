Huge Cinnamon Dalmatian

medium

# Some curve pools will not be able to remove liquidity single sided

## Summary
Some curve pools uses the remove_liquidity_one_coin function with uint256 but the notional code assumes it is always int128 which if the pool to be used is using uint256 then the removing single sided liquidity is impossible.
## Vulnerability Detail
First this is the interface that the Notional uses when removing single sided liquidity from curve:
```solidity
interface ICurve2TokenPool is ICurvePool {
    function remove_liquidity(uint256 amount, uint256[2] calldata _min_amounts) external returns (uint256[2] memory);
    function remove_liquidity_one_coin(uint256 _token_amount, int128 i, uint256 _min_amount) external returns (uint256);
}
```
so it uses uint256, int128 and uint256 as inputs. Let's see some of the pools that notional can potentially use. One can be the rETH-ETH pool for ETH leveraged vault. This is how the remove_liquidity_one_coin function is built:
```vyper
@external
@nonreentrant('lock')
def remove_liquidity_one_coin(token_amount: uint256, i: uint256, min_amount: uint256,
                              use_eth: bool = False, receiver: address = msg.sender) -> uint256:
```

as we can see it uses uint256, uint256 , uint256. In Notionals interface above it uses int128 for the  `I` variable but the actual pool uses uint256. This will fail because casting an interface input from int128 to uint256 is not supported in solidity. 
## Impact
Pool can be deployed without a problem. However, in production the removing single sided liquidity will not work. Also, Notional team states that every "potential" curve pool can be used and the above example is rETH-ETH which is a very good candidate for ETH strategy. Thus, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/Curve2TokenConvexVault.sol#L78-L84
## Tool used

Manual Review

## Recommendation
Make a flag that tells the contract that the pool uses remove_liquidity_one_coin with uint256 inputs. 