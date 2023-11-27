Huge Cinnamon Dalmatian

high

# Yield can be sandwiched

## Summary
During a reinvestment, the total claims of the pool increase, but the shares remain unaffected. Consequently, users who hold vault shares experience an increase in LP token claims on their shares. However, the reinvestment call is susceptible to being sandwiched. If a user deposits just before the reinvestment and exits immediately afterward, that user pockets the profit without being an actual vault shareholder. This action disrupts the yield of other vault shareholders.
## Vulnerability Detail
Let's consider a scenario where the total vault shares amount to 100 tokens, and there are 100 LP tokens in the vault. When the reinvestment manager triggers a reinvestment, buying an additional 10 LP tokens, Alice, an MEV expert, notices this transaction in the mempool. She swiftly enters the vault just before the reinvestment occurs, mints tokens at a 1:1 ratio, and immediately after the reinvestment function, Alice exits the vault, claiming the yield for herself. This maneuver adversely affects other vault holders, resulting in them receiving lower yields, while Alice gains an unfair advantage by being in the vault for only one block.

Moreover, Alice can scale up this attack significantly. By depositing a large number of tokens via flash loans, she can acquire a substantial share from the vault. As she holds the majority of the shares, Alice receives the bulk of the yield. Consequently, other vault shareholders receive minimal yield, significantly diminishing their returns.
## Impact
High since it can block the yield
## Code Snippet
https://github.com/sherlock-audit/2023-10-notional/blob/7aadd254da5f645a7e1b718e7f9128f845e10f02/leveraged-vaults/contracts/vaults/common/SingleSidedLPVaultBase.sol#L385-L411
## Tool used

Manual Review

## Recommendation
