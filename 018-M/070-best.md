Shallow Peanut Elk

medium

# Maximum number of tokens supported is incorrect

## Summary

Notional is unable to deploy the new leverage vault for a Composable Pool that has 5 stable tokens, rendering the protocol useless for such types of pools.

## Vulnerability Detail

The following code taken from Balancer's composable pool shows that a composable pool can support up to 6 tokens (5 stable tokens + 1 BPT)

https://github.com/balancer/balancer-v2-monorepo/blob/c7d4abbea39834e7778f9ff7999aaceb4e8aa048/pkg/pool-stable/contracts/ComposableStablePoolStorage.sol#L206

```solidity
File: ComposableStablePoolStorage.sol
204:     function _getMaxTokens() internal pure override returns (uint256) {
205:         // The BPT will be one of the Pool tokens, but it is unaffected by the Stable 5 token limit.
206:         return StableMath._MAX_STABLE_TOKENS + 1;
207:     }
```

https://github.com/balancer/balancer-v2-monorepo/blob/c7d4abbea39834e7778f9ff7999aaceb4e8aa048/pkg/pool-stable/contracts/StableMath.sol#L32

```solidity
File: StableMath.sol
25: library StableMath {
26:     using FixedPoint for uint256;
27: 
28:     uint256 internal constant _MIN_AMP = 1;
29:     uint256 internal constant _MAX_AMP = 5000;
30:     uint256 internal constant _AMP_PRECISION = 1e3;
31: 
32:     uint256 internal constant _MAX_STABLE_TOKENS = 5;
```

However, the Notional's leverage vault only supports up to a total of five (5) pool tokens. As a result, any composable pool with a total of 6 tokens is incompatible with the leverage vault.

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L40

```solidity
File: SingleSidedLPVaultBase.sol
40:     uint256 internal constant MAX_TOKENS = 5;
```

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/BalancerPoolMixin.sol#L89

```solidity
File: BalancerPoolMixin.sol
75:     constructor(NotionalProxy notional_, DeploymentParams memory params)
76:         SingleSidedLPVaultBase(notional_, params.tradingModule) {
77: 
78:         BALANCER_POOL_ID = params.balancerPoolId;
79:         (address pool, /* */) = Deployments.BALANCER_VAULT.getPool(params.balancerPoolId);
80:         BALANCER_POOL_TOKEN = IERC20(pool);
81: 
82:         // Fetch all the token addresses in the pool
83:         (
84:             address[] memory tokens,
85:             /* uint256[] memory balances */,
86:             /* uint256 lastChangeBlock */
87:         ) = Deployments.BALANCER_VAULT.getPoolTokens(params.balancerPoolId);
88: 
89:         require(tokens.length <= MAX_TOKENS);
90:         _NUM_TOKENS = uint8(tokens.length);
91: 
92:         TOKEN_1 = _NUM_TOKENS > 0 ? _rewriteWETH(tokens[0]) : address(0);
93:         TOKEN_2 = _NUM_TOKENS > 1 ? _rewriteWETH(tokens[1]) : address(0);
94:         TOKEN_3 = _NUM_TOKENS > 2 ? _rewriteWETH(tokens[2]) : address(0);
95:         TOKEN_4 = _NUM_TOKENS > 3 ? _rewriteWETH(tokens[3]) : address(0);
96:         TOKEN_5 = _NUM_TOKENS > 4 ? _rewriteWETH(tokens[4]) : address(0);
```

## Impact

Notional is unable to deploy the new leverage vault for a Composable Pool that has 5 stable tokens, rendering the protocol useless for such types of pools.

In addition, if the affected vaults cannot be used, it leads to a loss of revenue for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L40

https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/mixins/BalancerPoolMixin.sol#L89

## Tool used

Manual Review

## Recommendation

Consider supporting up to a total of 6 pool tokens to ensure that all composable pools work with the leverage vault.

```diff
- uint256 internal constant MAX_TOKENS = 5;
+ uint256 internal constant MAX_TOKENS = 6;
```

```diff
require(tokens.length <= MAX_TOKENS);
_NUM_TOKENS = uint8(tokens.length);

TOKEN_1 = _NUM_TOKENS > 0 ? _rewriteWETH(tokens[0]) : address(0);
TOKEN_2 = _NUM_TOKENS > 1 ? _rewriteWETH(tokens[1]) : address(0);
TOKEN_3 = _NUM_TOKENS > 2 ? _rewriteWETH(tokens[2]) : address(0);
TOKEN_4 = _NUM_TOKENS > 3 ? _rewriteWETH(tokens[3]) : address(0);
TOKEN_5 = _NUM_TOKENS > 4 ? _rewriteWETH(tokens[4]) : address(0);
+ TOKEN_6 = _NUM_TOKENS > 5 ? _rewriteWETH(tokens[5]) : address(0);

DECIMALS_1 = _NUM_TOKENS > 0 ? TokenUtils.getDecimals(TOKEN_1) : 0;
DECIMALS_2 = _NUM_TOKENS > 1 ? TokenUtils.getDecimals(TOKEN_2) : 0;
DECIMALS_3 = _NUM_TOKENS > 2 ? TokenUtils.getDecimals(TOKEN_3) : 0;
DECIMALS_4 = _NUM_TOKENS > 3 ? TokenUtils.getDecimals(TOKEN_4) : 0;
DECIMALS_5 = _NUM_TOKENS > 4 ? TokenUtils.getDecimals(TOKEN_5) : 0;
+ DECIMALS_6 = _NUM_TOKENS > 5 ? TokenUtils.getDecimals(TOKEN_6) : 0;
```