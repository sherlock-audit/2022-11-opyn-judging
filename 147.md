0x52

medium

# Adversary can DOS entire contract by withdrawing CRAB then puposely getting themselves blacklisted

## Summary

Circle blacklists any user that interacts with Tornado cash. Since CRAB withdraws are always processed in order an adversary can use this to DOS the entire contract. The contract sends USDC payment directly to the user requesting the withdraw, if the user is blacklisted then it will revert when trying to send payment to that user. Without a way to eject deposit/withdraws the only way to clear them is to fill them or dequeue them. Since a withdraw can't be filled for a USDC blacklisted user, the contract would be permanently DOS'd.

## Vulnerability Detail

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

The above code processes the withdraws in the order that they are submitted. If there is a single bad withdraw in which the recipient can't accept the USDC transfer then the contract would be permanently DOS'd. To get an address blacklisted for USDC it is as easy as interacting with Tornado Cash.

## Impact

Entire contract can be DOS'd by one bad withdraw

## Code Snippet

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L390-L418

## Tool used

Manual Review

## Recommendation

Consider using a censor resistant token for payment such as DAI or WETH. You can use the following fix if continuing to use USDC. If the sender cannot receive the payment, send it to a bypass contract to handle the withdraw:

            crabBalance[withdraw.sender] -= withdraw.amount;
            amountToSend = (withdraw.amount * _price) / 1e18;
    -       IERC20(usdc).transfer(withdraw.sender, amountToSend);
    +       try IERC20(usdc).transfer(withdraw.sender, amountToSend) {
    +       } catch {
    +           //send tokens to another contract to handle withdraws
    +       }