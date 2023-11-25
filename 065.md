Huge Cinnamon Dalmatian

medium

# Malicious reward investor role holder can manipulate price of vault shares

## Summary
The Reward Investor role bears the responsibility of selling the reward tokens to acquire more of the underlying tokens and subsequently re-liquidity provisions (reLP) them. As outlined by the protocol team, this role is tasked solely with these actions. However, an economic incentive may arise where manipulating the price per share of the vault token becomes advantageous for this role. In such cases, they might execute an airdrop of some reward tokens to the strategy, subsequently selling them. This strategic move aims to bolster the price of the vault token.
## Vulnerability Detail
Suppose the protocol team delegates the role of the reward investor to an external party, or let's consider a scenario where a bot automatically sells the rewards by utilizing "rewardToken.balanceOf(vault)" during execution. In such cases, a malicious user might exploit this system by sending reward tokens to the vault, thereby inflating the price of the vault share. This manipulation tactic could potentially be employed within the Notional lending and borrowing domain to liquidate someone or manipulate aspects of the market.
## Impact
If it is economically makes sense for the role holder to perform such thing then they can do that even in a single transaction. Although I am not sure how this can be to leveraged this for advantage it is still possible for bot to increase the price per share by purpose or in accident.  
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L385-L427
## Tool used

Manual Review

## Recommendation
Limit the maximum reward token sell amount such that the pps does not increases unexpectedly because of a bot accounting error or malicious bot owner.