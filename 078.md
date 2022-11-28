csanuragjain

medium

# DOS in user withdrawal

## Summary
User will not be able to withdraw full amount post deposit. This is due to the fact that `userDepositsIndex[msg.sender]` is never deleted while withdraw which cause array out of index error

## Note
Another impacted function is dequeueCrab function which suffers from same issue and in this case `userWithdrawsIndex[msg.sender][i - 1]` should be deleted once `withdraws[userWithdrawsIndex[msg.sender][i - 1]]` is deleted

## Vulnerability Detail
1. User deposits 1000 USDC using `depositUSDC` function

```solidity
function depositUSDC(uint256 _amount) external {
        ...
        usdBalance[msg.sender] = usdBalance[msg.sender] + _amount;
        deposits.push(Receipt(msg.sender, _amount));
        userDepositsIndex[msg.sender].push(deposits.length - 1);

        ...
    }
```

2. A simplified version will be :

```solidity
deposits[0]=Receipt(msg.sender, 1000)
userDepositsIndex[msg.sender][0]=deposits.length - 1 = 1-1 = 0
```

3. Now User decides to withdraw full 1000 USDC using `withdrawUSDC` function

```solidity
function withdrawUSDC(uint256 _amount) external {

...
uint256 lastIndexP1 = userDepositsIndex[msg.sender].length;
        for (uint256 i = lastIndexP1; i > 0; i--) {
...
if (r.amount > toRemove) {
...
}
else {
                toRemove -= r.amount;
                delete deposits[userDepositsIndex[msg.sender][i - 1]];
            }
}
...

}
```

4. Since in our case `r.amount = toRemove` so else part is executed which deletes the index 

```solidity
delete deposits[userDepositsIndex[msg.sender][1 - 1]] = delete deposits[userDepositsIndex[msg.sender][0]] = delete deposits[0]
```

5. Post this user deposits 10000 USDC which update the variables as:

```solidity
deposits[0]=Receipt(msg.sender, 10000)
userDepositsIndex[msg.sender][0]=0 (from step 2, remember this was not deleted in withdraw)
userDepositsIndex[msg.sender][1]=deposits.length - 1 = 1-1 = 0
```

6. User withdraw the full amount which executes below:

```solidity
function withdrawUSDC(uint256 _amount) external {

...
uint256 lastIndexP1 = userDepositsIndex[msg.sender].length;
        for (uint256 i = lastIndexP1; i > 0; i--) {
...
if (r.amount > toRemove) {
...
}
else {
                toRemove -= r.amount;
                delete deposits[userDepositsIndex[msg.sender][i - 1]];
            }
}
...

}
```

7. Since `userDepositsIndex[msg.sender].length` is 2 so loop run twice. 

8. In first loop run else part executes and this deletes deposits[0]
9. On second loop run again else part is executed which tries 

```solidity
delete deposits[userDepositsIndex[msg.sender][i - 1]]; = delete deposits[userDepositsIndex[msg.sender][0]]; = delete deposits[0] = error since deposits[0]  index does not exist (it was already deleted in step 8)
```

## Impact
User will not be able to withdraw the full deposited amount

## Code Snippet
https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L298
https://github.com/opynfinance/squeeth-monorepo/blob/main/packages/crab-netting/src/CrabNetting.sol#L341

## Tool used
Manual Review

## Recommendation
Delete the array index at userDepositsIndex[msg.sender][i - 1] as well