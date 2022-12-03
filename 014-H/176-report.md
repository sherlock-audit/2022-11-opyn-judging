dipp

high

# Users could receive 0 tokens for their deposits

## Summary

Users are able to withdraw most of the crab from a withdrawal receipt (as long as the user's ```crabBalance > minCrabAmount```) which could cause a rounding down issue when the amount of USDC to send to the user is calculated.

## Vulnerability Detail

When crab tokens are sent to users using the ```netAtPrice``` function in ```CrabNetting.sol```, the ```amountToSend``` is calculated as ```(withdraw.amount * _price) / 1e18```. Since ```_price``` is given in ```USDC``` which has 6 decimals, if ```withdraw.amount``` is very low then ```amountToSend``` could round down to 0. The amount of crab for a ```withdraws``` receipt is able to go below ```minCrabAmount``` when a user calls the ```dequeueCrab``` function since the function only checks that the user's ```crabBalance``` is above ```minCrabAmount```.

The calculations used to determine the USDC amount to send in ```withdrawAuction``` would only allow this to happen if the withdraws receipt has very little dust amount of crab.

This would only happen for USDC deposits if the price of crab in USDC is extremely high.

## Impact

Users could receive no USDC for their crab.

## Code Snippet

A user calls the ```dequeueCrab``` function in ```CrabNetting.sol``` to reduce the amount of crab in a receipt to be very low such that (withdraw.amount * priceOfCrabInUSDC) < 1e18.

[CrabNetting.sol:dequeueCrab#L323-L346](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L323-L346):
```solidity
    function dequeueCrab(uint256 _amount) external {
        require(!isAuctionLive, "auction is live");
        crabBalance[msg.sender] = crabBalance[msg.sender] - _amount;
        require(
            crabBalance[msg.sender] >= minCrabAmount || crabBalance[msg.sender] == 0,
            "remaining amount smaller than minimum, consider removing full balance"
        );
        // deQueue crab from the last, last in first out
        uint256 toRemove = _amount;
        uint256 lastIndexP1 = userWithdrawsIndex[msg.sender].length;
        for (uint256 i = lastIndexP1; i > 0; i--) {
            Receipt storage r = withdraws[userWithdrawsIndex[msg.sender][i - 1]];
            if (r.amount > toRemove) {
                r.amount -= toRemove;
                toRemove = 0;
                break;
            } else {
                toRemove -= r.amount;
                delete withdraws[userWithdrawsIndex[msg.sender][i - 1]];
            }
        }
        IERC20(crab).transfer(msg.sender, _amount);
        emit CrabDeQueued(msg.sender, _amount, crabBalance[msg.sender]);
    }
```

The ```netAtPrice``` function is then called by the owner and the receipt that was lowered by the user will not provide any USDC to the receipt's owner.

[CrabNetting.sol:netAtPrice#L353-L419](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L353-L419):
```solidity
    function netAtPrice(uint256 _price, uint256 _quantity) external onlyOwner {
        _checkCrabPrice(_price);
        uint256 crabQuantity = (_quantity * 1e18) / _price;
        require(_quantity <= IERC20(usdc).balanceOf(address(this)), "Not enough deposits to net");
        require(crabQuantity <= IERC20(crab).balanceOf(address(this)), "Not enough withdrawals to net");

        // process deposits and send crab
        uint256 i = depositsIndex;
        uint256 amountToSend;
        while (_quantity > 0) {
            Receipt memory deposit = deposits[i];
            if (deposit.amount == 0) {
                i++;
                continue;
            }
            if (deposit.amount <= _quantity) {
                // deposit amount is lesser than quantity use it fully
                _quantity = _quantity - deposit.amount;
                usdBalance[deposit.sender] -= deposit.amount;
                amountToSend = (deposit.amount * 1e18) / _price;
                IERC20(crab).transfer(deposit.sender, amountToSend);
                emit USDCDeposited(deposit.sender, deposit.amount, amountToSend, i, 0);
                delete deposits[i];
                i++;
            } else {
                // deposit amount is greater than quantity; use it partially
                deposits[i].amount = deposit.amount - _quantity;
                usdBalance[deposit.sender] -= _quantity;
                amountToSend = (_quantity * 1e18) / _price;
                IERC20(crab).transfer(deposit.sender, amountToSend);
                emit USDCDeposited(deposit.sender, _quantity, amountToSend, i, 0);
                _quantity = 0;
            }
        }
        depositsIndex = i;

        // process withdraws and send usdc
        i = withdrawsIndex;
        while (crabQuantity > 0) {
            Receipt memory withdraw = withdraws[i];
            if (withdraw.amount == 0) {
                i++;
                continue;
            }
            if (withdraw.amount <= crabQuantity) {
                crabQuantity = crabQuantity - withdraw.amount;
                crabBalance[withdraw.sender] -= withdraw.amount;
                amountToSend = (withdraw.amount * _price) / 1e18;
                IERC20(usdc).transfer(withdraw.sender, amountToSend);

                emit CrabWithdrawn(withdraw.sender, withdraw.amount, amountToSend, i);

                delete withdraws[i];
                i++;
            } else {
                withdraws[i].amount = withdraw.amount - crabQuantity;
                crabBalance[withdraw.sender] -= crabQuantity;
                amountToSend = (crabQuantity * _price) / 1e18;
                IERC20(usdc).transfer(withdraw.sender, amountToSend);

                emit CrabWithdrawn(withdraw.sender, withdraw.amount, amountToSend, i);

                crabQuantity = 0;
            }
        }
        withdrawsIndex = i;
    }
```

## Tool used

Manual Review

## Recommendation

Consider skipping the receipts if the amountToSend rounds down to 0 so that the user is able to withdraw the remainder of the tokens from the receipt.