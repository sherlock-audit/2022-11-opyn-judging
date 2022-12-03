thec00n

medium

# Zero amount deposits could make auction and swap functions unusable

## Summary
Deposits and withdrawals are lists that are processed sequentially. A new entry is added when a user deposits USDC or Crabv2 tokens. A malicious user could create many zero-amount deposits and withdrawals in a row which would cause the functions `netAtPrice()`, `depositAuction()` and `withdrawAuction()` to always fails because the block gas limit has been reached in the loops that process the deposit and withdraw lists. 

A similar issue exists for `userDepositsIndex` and `userWithdrawsIndex`. The array stores index to deposits and it only grows with every new deposit, and indexes are never deleted. A `withdrawUSDC()` or `dequeueCrab()` always runs through the entire list and theoretically with enough items in the list it could exceed the block gas limit.   

## Vulnerability Detail
A large number of zero-amount withdrawals added in a row could cause the loop at L693-L695 to exceed the block gas limit and always fail.  

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L691-L721

The other instances where this can occur are listed in the code snippet section.

## Impact
The auction and swap functions could become unusable. Users need to withdraw their funds, and a new version of the contracts needs to be deployed. 

## Code Snippet


https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L393-L396

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L582-L585

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L695-L698

## Tool used
Manual Review

## Recommendation
Currently it's possible to make zero amount deposits because `setMinUSDC()` and `setMinCrab()` is not set at contract creation. Even if it's set, a large number of zero amount deposits can created by repeately depositing and withdrawing the same amount if `toggleAuctionLive()` is not set. The current way of processing data strucures in while loops is not optimal as it carries the risk of hitting the block gas limit. At the very least a safe maximum iteration limit should be set in the contract or as function parameter that gurantees that the loops do not exceed the block gas limit.    
