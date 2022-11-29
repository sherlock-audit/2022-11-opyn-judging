koxuan

high

# depositAuction will always revert if to_send.eth is more than ethToFlashDeposit

## Summary
In depositAuction, `flashDeposit` is used to swap eth to sqth which will be sent to market maker orders who are buying sqth. However, if the to_send.eth is more than ethToFlashDeposit, no swap will take place. Hence, no sqth will be swapped and will cause a revert during the next step where sqth is being sent back to the order's user.

## Vulnerability Detail
In `depositAuction`, step 4 does a flashDeposit, which swaps eth for sqth. `to_send.eth <= p.ethToFlashDeposit` ensures that we don't send more eth than the flashDeposit. However, if to_send.eth is more than `p.ethToFlashDeposit`, it will not do the flashDeposit and therefore no eth is swapped to sqth.
```solidity
        // step 4
        Portion memory to_send;
        to_send.eth = address(this).balance - initEthBalance;
        if (to_send.eth > 0 && _p.ethToFlashDeposit > 0) {
            if (to_send.eth <= _p.ethToFlashDeposit) {
                // we cant send more than the flashDeposit
                ICrabStrategyV2(crab).flashDeposit{value: to_send.eth}(_p.ethToFlashDeposit, _p.flashDepositFee);
            }
        }
```
With no eth being swapped to sqth, step 5 will revert due to insufficient sqth.
```solidity
        // step 5
        to_send.sqth = IERC20(sqth).balanceOf(address(this));
        remainingToSell = to_send.sqth;
        for (uint256 j = 0; j < _p.orders.length; j++) {
            if (_p.orders[j].quantity < remainingToSell) {
                IERC20(sqth).transfer(_p.orders[j].trader, _p.orders[j].quantity);
                remainingToSell -= _p.orders[j].quantity;
                emit BidTraded(_p.orders[j].bidId, _p.orders[j].trader, _p.orders[j].quantity, _p.clearingPrice, true);
            } else {
                IERC20(sqth).transfer(_p.orders[j].trader, remainingToSell);
                emit BidTraded(_p.orders[j].bidId, _p.orders[j].trader, remainingToSell, _p.clearingPrice, true);
                break;
            }
        }
```

## Impact
eth is not being swapped for sqth, causing contract to not have sufficient sqth to send back to market maker order users and hence causing a revert.

## Code Snippet
[CrabNetting.sol#L543-L551](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L543-L551)
[CrabNetting.sol#L553-L566](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L553-L566)
## Tool used

Manual Review

## Recommendation
Add a else clause to handle `to_send.eth >  p.ethToFlashDeposit`
```solidity
  if (to_send.eth <= _p.ethToFlashDeposit) {
      // we cant send more than the flashDeposit
      ICrabStrategyV2(crab).flashDeposit{value: to_send.eth}(_p.ethToFlashDeposit, _p.flashDepositFee);
  }
  else{
      ICrabStrategyV2(crab).flashDeposit{value: _p.ethToFlashDeposit}(_p.ethToFlashDeposit, _p.flashDepositFee);
  }
```