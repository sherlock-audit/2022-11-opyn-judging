koxuan

medium

# zero can be sent in netAtPrice, causing loss to depositor or withdrawer

## Summary
There is no zero check to check for rounding error in netAtPrice, allowing zero to be sent even though usdc or crab is deducted from user balances.

## Vulnerability Detail
calculation of crab to send to depositor
```solidity
    _quantity = _quantity - deposit.amount;
    usdBalance[deposit.sender] -= deposit.amount;
    amountToSend = (deposit.amount * 1e18) / _price;
```

calculation usdc to send to withdrawer
```solidity
    crabQuantity = crabQuantity - withdraw.amount;
    crabBalance[withdraw.sender] -= withdraw.amount;
    amountToSend = (withdraw.amount * _price) / 1e18;
```

In both scenarios, amountToSend can be zero even though usdBalance and crabBalance have been deducted. This will go through, causing user to lose usdc or crab. 

## Impact
User can lose usdc or crab due to rounding error.
## Code Snippet
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L372
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L381
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L400
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L410

## Tool used

Manual Review

## Recommendation
Consider handling zero amount scenario.
