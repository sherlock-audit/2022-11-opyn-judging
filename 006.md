thec00n

medium

# Orders from other market makers can be invalidated

## Summary
The `checkOrder()` function performs verification of pre-signed orders. This function allows anyone to set the status of an order as used by storing the nonce contained in the order. Orders and their respective nonce can only be used once.  

## Vulnerability Detail
The `_useNonce()` function is called as called as part of the `checkOrder()` function. It checks that the nonce of a trader has not already been used, marks the nonce as used and performs other order verification checks. Orders and their respective nonce are also checked by the same implementation as part of the auction functions `withdrawAuction()` and `depositAuction()`. An order that has been invalidated once can not be used anymore and by calling `checkOrder()` any user can invalidate existing orders. 

## Impact
A malicious user could perform a grieving attack and invalidate any presigned orders by monitoring the mempool and front run any orders that are submitted to `withdrawAuction()` and `depositAuction()` and send them to `checkOrder()`. One invalidated order can cause the auction functions to fail.

## Code Snippet
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L447-L476

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L756-L759

## Tool used
Manual Review

## Recommendation
1.) Change the `checkOrder()` and `_checkOrder()` to a view function 
2.) Remove `_useNonce()` from `_checkOrder()`
3.) Use `_useNonce()` and `_checkOrder()` in `withdrawAuction()` and `depositAuction()`