---
eip: 
title: Standardized Time Locks
description: A standard for conveying the date upon which a time-locked system becomes unlocked
author: Thanh Trinh (@thanhtrinh2003) <thanh@revest.finance>, Joshua Weintraub (@jhweintraub) <josh@revest.finance>, Rob Montgomery (@RobAnon) <rob@revest.finance>
discussions-to: 
status: Draft
type: Standards Track
created: 2023-06-05
requires: 165
---

## Abstract

This EIP defines a standardized method to communicate the date on which a time-locked system will become unlocked. This allows for the determination of maturities for a wide variety of asset classes and increases the ease with which these assets may be valued.

## Motivation

Time-locks are ubiquitous, yet no standard on how to determine the date upon which they unlock exists. Time-locked assets experience theta-decay, where the time remaining until they become unlocked dictates their value. Providing a universal standard to view what date they mature on allows for improved on-chain valuations of the rights to these illiquid assets, particularly useful in cases where the rights to these illiquid assets may be passed between owners through semi-liquid assets such as ERC-721s or ERC-1155s.  

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

All EIP-7777 Standardized Time Locks locks MUST implement [ERC-165](./eip-165.md) interface detection.

#
```solidity
// SPDX-License-Identifier: CC0-1.0

pragma solidity ^0.8.0;

interface TimelockMaturity {
    /
     * @notice      This function returns the timestamp that the time lock specified by `id` unlocks at
     * @param       id The identifier which describes a specific time lock
     * @return      maturity The timestamp of the time lock when it unlocks
     */
    function getMaturity(bytes32 id)
        external
        view
        returns (uint256 maturity);

}
```

For singleton implementations on fungible assets, values passed to `id` SHOULD be ignored, and queries to such implementations should pass in `0x0` 

## Rationale

### Universal Maturities on Locked Assets

Locked Assets have become increasingly popular and used in different parts of defi, such as yield farming and vested escrow concept. This has increased the need to formalize and define an universal interface for all these timelocked assets.

### Valuation of Locked Assets via the Black-Scholes Model

 Locked Assets cannot be valued normally since the value of these assets can be varied through time and many other different factors throughout the locking time. For instance, The Black-Scholes Model or Black-Scholes-Merton model is an example of a suitable model to estimate the theoritical value of asset with the consideration of impact of time and other potential risks. 

$$ C = N(d_1)S_t - N(d_2)Ke^{-ert} $$

$$ \text {where } \small d_1 = \frac{\ln\frac{S_t}{K} + (r + \frac{\sigma^2}{2})}{\sigma \sqrt t} \text{ and } d2 = d1 - \sigma \sqrt t$$

 $ \small C =  \text {call option price} $
 $ \small N =  \text {CDF of the normal distribution} $

-  $ \small N =  \text {CDF of the normal distribution} $

-  $ \small S_t =  \text {spot price of an asset} $

-  $ \small K =  \text {strike price} $

-  $ \small r =  \text {risk-free interest rate} $

-  $ \small t =  \text {time to maturity} $

-  $ \small \sigma =  \text {volatility of the asset}$

Time to maturity plays an important role in evaluating the price of timelocked assets, thus the demand to have a common interface for retrieving the data is inevitable. 

## Backwards Compatibility

This standard can be implemented as an extension to [ERC-721](./eip-721.md) and/or [ERC-1155](./eip-1155.md) tokens with time-locked functionality, many of which can be retrofitted with a designated contract to determine the point at which their time locks release. 

## Reference Implementation

### Locked ERC-20 implementation
```solidity
// SPDX-License-Identifier: CC0-1.0

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract LockedERC20ExampleContract implements TimelockMaturity{
    ERC20 public token;
    uint256 public override totalLocked;

    //Timelock struct
    struct TimeLock {
        address owner;
        uint256 amount;
        uint256 maturity;
        bytes32 lockId;
    }

    //maps lockId to balance of the lock
    mapping(bytes32 => TimeLock) public override idToLock;    

    function constructor(
        address _owner,
        address _token,
        string memory _name,
        string memory _symbol
    ) public {
        token = ERC20(_token);
        totalLocked = 0;
    }

    //Maturity is not appropriate
    error LockPeriodOngoing();
    error InvalidReceiver();
    error TransferFailed();

    /// @dev Deposit tokens to be locked in the requested locking period
    /// @param amount The amount of tokens to deposit
    /// @param lockingPeriod length of locking period for the tokens to be locked
    function deposit(uint256 amount, uint256 lockingPeriod) public returns (bytes32 lockId) {
        uint256 maturity = block.timestamp + lockingPeriod;
        lockedId = keccack256(abi.encoded(msg.sender, amount, maturity))

        TimeLock memory newLock = TimeLock(msg.sender, amount, maturity, lockedId);

        totalLocked += amount;

        if (!token.transferFrom(msg.sender, address(this), amount)) {
            revert TransferFailed();
        }

        lockedId = keccack256(abi.encoded(msg.sender, amount, maturity));
    }

    /// @dev Withdraw tokens in the lock after the end of the locking period
    /// @param lockId id of the lock that user have deposited in
    function withdraw(bytes32 lockId) public {
        TimeLock lock = idToLock[lockId];

        address owner =  lock.owner;
        uint256 currentMaturity = lock.maturity;
        uint256 amount = lock.amount;

        if (msg.sender != owner) {
            revert InvalidReceiver();
        }

        if (block.timestamp > currentMaturity) {
            revert LockPeriodOngoing();
        }

        totalLocked -= amount;

        if (!token.transfer(msg.sender, amount)) {
            revert TransferFailed();
        }
    }

    function getMaturity(bytes32 id) external view returns (uint256 maturity) {
        return idToLock[id].maturity;
    }
}

```

## Security Considerations

### Extendable Time Locks

Users or developers should be aware of potential extendable timelocks, where the returned timestamp can be modified through protocols. Users or protocols should check the timestamp carefully before trading or lending with others.


## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
