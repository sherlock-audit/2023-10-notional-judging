Wonderful Mulberry Orangutan

medium

# Functions can reach block gas-limit via unbounded operations

## Summary

Smart contract vulnerabilities, particularly related to gas limit issues, can have severe consequences. One critical vulnerability arises when functions demand more gas than the block limit allows, commonly occurring in loops that iterate over dynamic data structures.

## Vulnerability Detail

Certain smart contract functions are at risk due to inadequate size validation of arrays, particularly in loops iterating over dynamic data structures. Neglecting to check input array sizes before execution can lead to computational requirements exceeding the block gas limit, resulting in transaction failures and state rollback. This vulnerability is especially crucial in decentralized finance (DeFi) applications where precise function execution is vital.

## Impact

Gas limit vulnerabilities extend beyond transaction failures, potentially causing funds locking, contract freezing, operational disruptions, and broader economic consequences for users and the ecosystem.

1. **Funds Locking:** Users may struggle to access or withdraw funds, leading to financial losses and trust erosion.

2. **Contract Freezing:** Gas limit failures can freeze the contract's state, affecting DApps relying on its functionality.

3. **Operational Disruption:** Disruptions in DeFi protocols may inconvenience users and harm the project's reputation.

4. **Economic Consequences:** Gas-related failures can have economic repercussions for users, investors, and the ecosystem.

## Code Snippet

- [StrategyUtils.sol 27-55](https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L27#L55)
- [StrategyUtils.sol 59-87](https://github.com/sherlock-audit/2023-10-notional/blob/main/leveraged-vaults/contracts/vaults/common/StrategyUtils.sol#L59#L87)

## Tool Used

Manual Review

## Recommendation

To strengthen smart contracts against gas limit vulnerabilities and mitigate their impact, consider implementing the following measures:

1. **Check Array Length:**
   - Validate array lengths using the `require` statement before iterating to prevent exhaustion.

   ```solidity
   function executeDepositTrades(uint256[] memory amounts) external {
       require(amounts.length <= MAX_ARRAY_LENGTH, "Array length exceeds maximum");
       // rest of the function
   }
   ```

2. **Limit Iteration:**
   - Utilize a `for` loop with proper indexing to iterate over array elements, ensuring bounds are respected.

   ```solidity
   function executeRedemptionTrades(uint256[] memory exitBalances) external {
       require(exitBalances.length <= MAX_ARRAY_LENGTH, "Inner array length exceeds maximum");
       // rest of the loop body
   }
   ```

3. **Gas Limit Consideration:**
   - Be aware of gas limits for each Ethereum block. For extensive computations, consider breaking down tasks into smaller transactions.