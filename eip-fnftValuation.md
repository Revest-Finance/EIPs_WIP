---
eip: TBD
title: Quantifiable-Value FNFTs
description: An interface for building financial-NFTs with a measurable value
author: Josh Weintraub (@jhweintraub) <josh@revest.finance>, Rob Montgomery (@RobAnon) <rob@revest.finance>
discussions-to: 
status: Draft
type: Standards Track
category: ERC
created: 2023-07-26
requires: 721, 1155, 5615
---

## Abstract

This proposal defines an extension interface to value [ERC-721](./eip-721.md) and/or [ERC-1155](./eip-1155.md) tokens which represent financial positions. 

## Motivation

As DeFi continues to expand into new use-cases, the complexity of representing various financial positions can no longer be accomplished solely with fungible [ERC-20](./eip-20.md) tokens. New financial NFTs (FNFTs) have no agreed upon standard for calculating their value on-chain. These financial positions tokenized as NFTs which cannot be valued cannot be used in DeFi. As a result a new standard is needed to be able to universally value these increasingly-complex financial instruments.

Examples of FNFTs include: Staking FNFTs, Vesting FNFTs, Options, complex derivatives, etc.

However, if these NFTs are to be utilized on secondary markets like NFT-exchanges or NFT-lending platforms, then having a uniform standard to value these is necessary to create an efficient market. Existing valuing schemes are ad-hoc and calculated off-chain with no transparency for the buyers. 

Example: A Staking NFT with a fixed unlock-time and fixed-amount of tokens is being sold on a secondary exchange. As the unlock date approaches the discount for the locked-tokens should decrease towards zero. Valuing this instrument constantly can be cumbersome and return incorrect information if the user is inexperienced. A unified interface to return a canonical value simplifies both price-discovery and development of secondary markets.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

```
interface IValuableFNFT {
    // @notice      This function MUST return whether the asset which the FNFT is valued in terms of
    // @param       id  The token id of which to check the existence
    // @return      The address of the ERC-20 token to value the FNFT in.
    function getAsset(uint id) external view returns (address);


    // @notice      This function MUST return the number of tokens with a given id. If the token id does not exist, it MUST return 0.
    // @param       id The token id of which to fetch information
    // @return      Suggested market value of the FNFT in specified in units of "asset"
    function getBalance(uint id) external view returns (uint);
}
```

If the underlying NFT contract supports [ERC-1155](./eip-1155.md), it MUST also implement the extension [ERC-5615](./eip-5615.md) 1155Supply. `getBalance()` should therefore return the value of ONE of the FNFTs for a given `id`, not the entire supply.

Users MAY choose to implement a function which returns an array of addresses and uints in cases where the FNFT would be worth multiple-assets, but is optional.

If the underlying asset is ETH, then the address `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` should be used in its place.

Implementations MAY support ERC-165 interface discovery, but consumers MUST NOT rely on it.

### Reference Implementation

```
import "@openzeppelin/openzeppelin-contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/openzeppelin-contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/openzeppelin-contracts/interfaces/IERC20.sol";

contract ValuableNFT is ERC721, ReentrancyGuard {
    address public immutable asset;
    uint public supply;
    mapping(uint => lock) locks;

    struct lock {
        uint asset;
        uint amount;
        uint creationTime;
        uint duration;
    }

    constructor() ERC721("", "") {
    }

    function mintTimeLockedFNFT(address asset, uint amount, uint duration) external payable {
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        _safeMint(msg.sender, supply);
        locks[supply] = lock(asset, amount, block.timestamp, duration);
        supply++;
    }

    function withdraw(uint id) external nonReentrant {
        require(msg.sender == _ownerOf(id));
        
        lock memory _lock = locks[id];
        require(block.timestamp >= _lock.creationTime + _lock.duration);

        _burn(id);
        delete locks[id];

        IERC20(_lock.asset).transfer(msg.sender, _lock.amount);
    }

    function getAsset(uint id) external view returns (address) {
        return locks[id].asset;
    }

    function getValue(uint id) external view returns (uint) {
        lock memory _lock = locks[id];
        if (block.timestamp >= _lock.creationTime + _lock.duration) return _lock.amount;

        uint timePassed = block.timestamp - _lock.creationTime;

        //Value is calculated on a linear vesting schedule over the length of lock
        return amount * timePassed / _lock.duration;
    }
}
```

## Rationale

`ERC-1155 Supply` should be used since many users may hold an FNFT of the same ID in different amounts. A user holding `>1` of an FNFT can multiply the returned price by their held-supply after calling `balanceOf(address,uint)` on the contract.

`uint` was chosen as it can be easily casted to a `bytes32` when applicable.

## Backwards Compatibility

This standard can be implemented as an extension to [ERC-721](./eip-721.md) and/or [ERC-1155](./eip-1155.md) tokens without breaking compatibility. In cases where an existing token does not implement the standard, a wrapper contract can be built on-top which makes external calls to the underlying FNFT contract.

## Security Considerations

Developers should be aware of potential price-manipulation bugs which may cause security issues for protocols which use the standard for valuating tokens listed on their platform. An example would be NFT lending, where a vulnerable implementation allows a user to overvalue their NFT and borrow more funds than their NFT is worth as collateral.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
