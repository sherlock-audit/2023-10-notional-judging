Huge Cinnamon Dalmatian

medium

# Some curve pools can not be used as a single sided strategy

## Summary
For single sided curve strategy contest docs says that any curve pool should be ok to be used. However, some pools are not adaptable to the Curve2TokenPoolMixin abstract contract. 
## Vulnerability Detail
The contract assumes the existence of three types of pool scenarios. In one scenario, the pool address itself serves as the LP token. In another scenario, the LP token is obtained by querying the lp_token() or token() function. However, in some cases where potential integration with Notional is possible, certain pools lack a direct method to retrieve the underlying LP token. For instance, the sETH-ETH pool, which presents a promising option for a single-sided strategy in ETH vaults, does not offer public variables to access the underlying LP token. Although the pool contract contains the token variable that returns the LP token of the pool, this variable is not publicly accessible. Consequently, in such cases, querying the LP token directly from the pool is not feasible, and it becomes necessary to provide the LP token address as input.

Here the example contracts where neither of the options are available:
sETH-ETH: https://etherscan.io/address/0xc5424b857f758e906013f3555dad202e4bdb4567
hBTC-WBTC: https://etherscan.io/address/0x4ca9b3063ec5866a4b82e437059d2c43d1be596f

## Impact
All curve pools that can be used as a single sided strategy for notional leveraged vaults considered to be used but some pools are not adaptable thus I will label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/curve/Curve2TokenPoolMixin.sol#L73-L78
## Tool used

Manual Review

## Recommendation
Curve contracts are pretty different and it is very hard to make a generic template for it. I would suggest make the LP token also an input and remove the necessary code for setting it in the constructor. Since the owner is trusted this should not be a problem. 