0xSmartContract

medium

# Signature malleability not protected against

## Summary
OpenZeppelin has a vulnerability in versions lower than 4.7.3, which can be exploited by an attacker. The project uses a vulnerable version


## Vulnerability Detail
All of the conditions from the advisory are satisfied: the signature comes in a single bytes argument, ECDSA.recover() is used, and the signatures themselves are used for replay protection checks https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-4h98-2769-gh6h


## Impact
If a user calls `_checkOrder()`, notices a mistake, then calls `_checkOrder()` again, an attacker can use signature malleability to re-submit the first change request, as long as the old request has not expired yet.

The wrong, potentially now-malicious, address will be the valid change recipient, which could lead to the loss of funds (e.g. the attacker attacked, the user changed to another compromised address, noticed the issue, then changed to a whole new account address, but the attacker was able to change it back and withdraw the funds to the unprotected address).

## Code Snippet
[CrabNetting.sol#L471](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/src/CrabNetting.sol#L471)

```js
src/CrabNetting.sol:
  454       */
  455:     function _checkOrder(Order memory _order) internal {
  456:         _useNonce(_order.trader, _order.nonce);
  457:         bytes32 structHash = keccak256(
  458:             abi.encode(
  459:                 _CRAB_NETTING_TYPEHASH,
  460:                 _order.bidId,
  461:                 _order.trader,
  462:                 _order.quantity,
  463:                 _order.price,
  464:                 _order.isBuying,
  465:                 _order.expiry,
  466:                 _order.nonce
  467:             )
  468:         );
  469: 
  470:         bytes32 hash = _hashTypedDataV4(structHash);
  471:         address offerSigner = ECDSA.recover(hash, _order.v, _order.r, _order.s);
  472:         require(offerSigner == _order.trader, "Signature not correct");
  473:         require(_order.expiry >= block.timestamp, "order expired");
  474:     }

```


[package.json#L4](https://github.com/sherlock-audit/2022-11-opyn/blob/main/crab-netting/lib/openzeppelin-contracts/package.json#L4)

```js
lib/openzeppelin-contracts/package.json:
  1  {
  2:   "name": "openzeppelin-solidity",
  3:   "description": "Secure Smart Contract library for Solidity",
  4:   "version": "4.7.0",

```

## Tool used

Manual Review

## Recommendation

Change to version 4.7.3

