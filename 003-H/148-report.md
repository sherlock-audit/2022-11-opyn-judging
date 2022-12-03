0x52

medium

# Adverary can DOS contract by making a large number of deposits/withdraws then removing them all

## Summary

When a user dequeues a withdraw or deposit it leaves a blank entry in the withdraw/deposit. This entry must be read from memory and skipped when processing the withdraws/deposits which uses gas for each blank entry. An adversary could exploit this to DOS the contract. By making a large number of these blank deposits they could make it impossible to process any auction. 

## Vulnerability Detail

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

The code above processes deposits in the order they are submitted. An adversary can exploit this by withdrawing/depositing a large number of times then dequeuing them to create a larger number of blank deposits. Since these are all zero, it creates a fill or kill scenario. Either all of them are skipped or none. If the adversary makes the list long enough then it will be impossible to fill without going over block gas limit.

## Impact

Contract can be permanently DOS'd

## Code Snippet

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L362-L386

## Tool used

Manual Review

## Recommendation

Two potential solutions. The first would be to limit the number of deposits/withdraws that can be processed in a single netting. The second would be to allow the owner to manually skip withdraws/deposits by calling an function that increments depositsIndex and withdrawsIndex.