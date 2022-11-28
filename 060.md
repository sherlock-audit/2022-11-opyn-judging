thec00n

medium

# Used orders or revoked token authorizations can cause `withdrawAuction` and `depositAuction` to fail

## Summary
The owner must ensure that all orders are valid before submitting an auction, as a single order failure can revert an entire auction. The current implementation allows a market maker to invalidate their order by front-running an auction transaction, causing the auction to fail. Other ways to cause the auction functions to fail are listed below.  

## Vulnerability Detail 
A market maker can invalidate their order when `withdrawAuction()` and `depositAuction()` is submitted from the owner by:

- setting the nonce of their order as used by calling `setNonceTrue()` or by calling `checkOrder()` and setting the nonce of orders as used (see https://github.com/sherlock-audit/2022-11-opyn-thec00n/issues/1).
- By revoking permissions to transfer tokens for the `CrabNetting` contract or transferring required tokens from the trading account so that the transfer fails.  

Large user withdrawals could also occur right before the auction is submitted which could could cause the auction functions to fail. 

## Impact
A malicious market maker or user could perform a griefing attack and repeatedly cause auctions to fail. 

## Code Snippet
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L756-L759

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L507-L524

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L645-L656


## Tool used
Manual Review

## Recommendation
- The function `_useNonce()` should not fail when the nonce has been used. Instead, return a flag with the status of the nonce update. Orders that already have used nonces should be skipped from processing in `withdrawAuction()` and `depositAuction()`.
- The auction functions should check if the market maker has sufficient balance and if the `CrabNetting` contract is authorized to successfully perform the transfer from a related order. Insufficient permissions or funds from users that sign orders should also be skipped. 
- Require that auction `isAuctionLive` is set to true for `withdrawAuction()` and `depositAuction()` so withdrawals can not occur during auctions. 