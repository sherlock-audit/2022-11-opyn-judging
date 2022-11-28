thec00n

medium

# User withdrawals are dependent on admin actions

## Summary
Users can deposit USDC and Crabv2 tokens at any time, but there are limitations around withdrawals. Users could have permanently locked up their funds if specific owner actions are not triggered. 

## Vulnerability Detail
The owner can call `toggleAuctionLive()` and prevent any withdrawals from occurring. User withdrawals are only enabled again when the owner calls `withdrawAuction()` or `depositAuction()`. If the owner loses their key or becomes malicious and never calls these functions, then the users have no way of withdrawing their funds. 

## Impact
Users could get their funds locked up in the `Netting` contract without a way to withdraw them again.

## Code Snippet
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L276-L283

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L321-L327

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L223-L226

## Tool used
Manual Review

## Recommendation
Lock up times are necessary for the system to work but users should always be able to withdraw their funds eventually without any dependecy of the owner.  When users deposit tokens, a meaningful expiry timestamp should be set by the contract. Before the expiry deposits are locked and the funds can be used during auctions. After expiry deposits are skipped and users can withdraw them at any time. 
