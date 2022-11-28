hyh

medium

# Unsafe transfer is used for an external upgradable token (USDC)

## Summary

Unsafe transfer guarantees the transfer was fully complete only when it is implemented to fail whenever transfers can't be processed fully. Not all ERC20 implementations behave that way, some only return the success flag.

Due to that while having unsafe transfer for protocol controlled tokens produces no issues, using it for any external controlled token poses a vulnerability as long as this token implementation do not always fail for unsuccessful transfer, returning false instead, or a token can be upgraded to start behaving this way.

## Vulnerability Detail

As USDC is an upgradable contract, there is no guarantees that the next implementation will be failing instead of only returning bool success value.

For success flag case the CrabNetting accounting will become broken as unsuccessful transfers will be deemed completed and their records removed, which de facto will mean permanent freeze of funds as manual reconciliation can become unfeasible quick enough.

I.e. unsafe transfer means that if USDC be upgraded to not always fail during unsuccessful transfers for any reason, the CrabNetting will start freezing user funds. This will be permanent freezes as no rescue function is implemented as of now.

## Impact

Permanent user funds freeze takes place when a token fails to transfer the funds, but the protocol deems the operation as successful and removes the user's entry from internal accounting. Due to prerequisites setting the severity to be medium.

## Code Snippet

netAtPrice() uses `transfer` to return USDC:

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L389-L419

```solidity
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

withdrawAuction() similarly uses `transfer`:

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L687-L720

```solidity
        // step 5 pay all withdrawers and mark their withdraws as done
        uint256 remainingWithdraws = _p.crabToWithdraw;
        uint256 j = withdrawsIndex;
        uint256 usdcAmount;
        while (remainingWithdraws > 0) {
            Receipt memory withdraw = withdraws[j];
            if (withdraw.amount == 0) {
                j++;
                continue;
            }
            if (withdraw.amount <= remainingWithdraws) {
                // full usage
                remainingWithdraws -= withdraw.amount;
                crabBalance[withdraw.sender] -= withdraw.amount;

                // send proportional usdc
                usdcAmount = (((withdraw.amount * 1e18) / _p.crabToWithdraw) * usdcReceived) / 1e18;
                IERC20(usdc).transfer(withdraw.sender, usdcAmount);
                emit CrabWithdrawn(withdraw.sender, withdraw.amount, usdcAmount, j);
                delete withdraws[j];
                j++;
            } else {
                withdraws[j].amount -= remainingWithdraws;
                crabBalance[withdraw.sender] -= remainingWithdraws;

                // send proportional usdc
                usdcAmount = (((remainingWithdraws * 1e18) / _p.crabToWithdraw) * usdcReceived) / 1e18;
                IERC20(usdc).transfer(withdraw.sender, usdcAmount);
                emit CrabWithdrawn(withdraw.sender, remainingWithdraws, usdcAmount, j);

                remainingWithdraws = 0;
            }
        }
        withdrawsIndex = j;
```

Same for depositUSDC() and withdrawUSDC():

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L257-L264

```solidity
    /**
     * @notice queue USDC for deposit into crab strategy
     * @param _amount USDC amount to deposit
     */
    function depositUSDC(uint256 _amount) external {
        require(_amount >= minUSDCAmount, "deposit amount smaller than minimum OTC amount");

        IERC20(usdc).transferFrom(msg.sender, address(this), _amount);
```

https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L274-L304

```solidity
    /**
     * @notice withdraw USDC from queue
     * @param _amount USDC amount to dequeue
     */
    function withdrawUSDC(uint256 _amount) external {
        require(!isAuctionLive, "auction is live");

        usdBalance[msg.sender] = usdBalance[msg.sender] - _amount;
        ...
        IERC20(usdc).transfer(msg.sender, _amount);

        emit USDCDeQueued(msg.sender, _amount, usdBalance[msg.sender]);
    }
```

## Tool used

Manual Review

## Recommendation

Consider using `safeTransfer` for USDC and any other externally managed tokens.
