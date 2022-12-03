Jeiwan

medium

# Missing tokens spending approval from a trader can impair the auction

## Summary
Missing tokens spending approval from a trader can impair the auction
## Vulnerability Detail
When receiving WETH (in [depositAuction](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L512-L520)) and oSQTH (in [withdrawAuction](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L649-L651)) from traders the transaction will be reverted if one of the traders hasn't set up token spending approval for the `CrabNetting` contract. Since it's the contract owner who executes the functions and not traders, traders are not directly forced to set up approvals.

## Impact
In case a trader willing to buy a big amount of oSQTH hasn't approved spending WETH to the `CrabNetting` contract, settling traders might be not possible until the trader's order is replaced with multiple smaller orders. In case there are more orders in the list, they won't be used to replace the trader who hasn't given an approval due to a revert.

## Code Snippet
[CrabNetting.sol#L507-L523](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L507-L523):
```solidity
for (uint256 i = 0; i < _p.orders.length; i++) {
    require(_p.orders[i].isBuying, "auction order not buying sqth");
    require(_p.orders[i].price >= _p.clearingPrice, "buy order price less than clearing");
    _checkOrder(_p.orders[i]);
    if (_p.orders[i].quantity >= remainingToSell) {
        IWETH(weth).transferFrom(
            _p.orders[i].trader, address(this), (remainingToSell * _p.clearingPrice) / 1e18
        );
        remainingToSell = 0;
        break;
    } else {
        IWETH(weth).transferFrom(
            _p.orders[i].trader, address(this), (_p.orders[i].quantity * _p.clearingPrice) / 1e18
        );
        remainingToSell -= _p.orders[i].quantity;
    }
}
```

[CrabNetting.sol#L643-L654](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L643-L654):
```solidity
for (uint256 i = 0; i < _p.orders.length && toPull > 0; i++) {
    _checkOrder(_p.orders[i]);
    require(!_p.orders[i].isBuying, "auction order is not selling");
    require(_p.orders[i].price <= _p.clearingPrice, "sell order price greater than clearing");
    if (_p.orders[i].quantity < toPull) {
        toPull -= _p.orders[i].quantity;
        IERC20(sqth).transferFrom(_p.orders[i].trader, address(this), _p.orders[i].quantity);
    } else {
        IERC20(sqth).transferFrom(_p.orders[i].trader, address(this), toPull);
        toPull = 0;
    }
}
```

## Tool used
Manual Review

## Recommendation
When receiving tokens from traders, consider checking approved limits and skipping orders from traders who haven't approved an enough amount–this will allow to use the other orders to successfully execute an auction.