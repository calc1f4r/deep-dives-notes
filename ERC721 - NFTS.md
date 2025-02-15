## Bugs due to `onerc721recieved`

In Solidity's ERC721 standard, functions like `safeMint`, `safeTransferFrom`, and `safeTransfer` are designed to ensure that tokens are only transferred to addresses capable of handling them. These functions interact with the `onERC721Received` function in recipient contracts to confirm that the recipient can accept ERC721 tokens. However, this design can inadvertently introduce reentrancy vulnerabilities if the recipient is a contract with custom dangerous logic in `function onERC721Received(address, address, uint256, bytes memory) public virtual returns (bytes4)` function selector.

Check Implementation [here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L159C1-L162C6).

```solidity
    function _safeMint(address to, uint256 tokenId, bytes memory data) internal virtual {
        _mint(to, tokenId);
        ERC721Utils.checkOnERC721Received(_msgSender(), address(0), to, tokenId, data);
    }
```

```solidity
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data) public virtual {
        transferFrom(from, to, tokenId);
        ERC721Utils.checkOnERC721Received(_msgSender(), from, to, tokenId, data);
    }
```

```solidity
    function _safeTransfer(address from, address to, uint256 tokenId) internal {
        _safeTransfer(from, to, tokenId, "");
    }
    function _safeTransfer(address from, address to, uint256 tokenId, bytes memory data) internal virtual {
        _transfer(from, to, tokenId);
        ERC721Utils.checkOnERC721Received(_msgSender(), from, to, tokenId, data);
    }
```

The Selector

```solidity
    function checkOnERC721Received(
        address operator,
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) internal {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(operator, from, tokenId, data) returns (bytes4 retval) {
                if (retval != IERC721Receiver.onERC721Received.selector) {
                    revert IERC721Errors.ERC721InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert IERC721Errors.ERC721InvalidReceiver(to);
                } else {
                    assembly ("memory-safe") {
                        revert(add(32, reason), mload(reason))
                    }
                }
            }
        }
    }
}
```

### 🚀 Target being not able to Retrieve NFTs because of no Implementation of `onerc721recieved`

As discussed above, if functions like `safemint`, `safetransfer`, and `safetransferFrom` are used to mint or transfer tokens to an address that is a contract, the contract must have a onERC721Received selector to ensure it accepts the NFT for the transfer to be successful.

```solidity
    function onERC721Received(address, address, uint256, bytes memory) public virtual returns (bytes4) {
        return this.onERC721Received.selector;
    }
```

If it doesn't have the `onERC721Received` selector, it won't be able to hold the transferred NFTs.

Example: In the [arkproject](https://codehawks.cyfrin.io/c/2024-07-ark-project/results?lt=contest&sc=reward&sj=reward&page=1&t=report), a protocol that allows you to transfer digital assets like ERC721 and ERC1155 tokens between L1 and StarkNet L2 using an escrow contract, this issue was encountered.

In the arkproject, there was a function called `withdrawtokens` to withdraw `ERC721` tokens received from L2. This function attempts to transfer tokens from the escrow which internally calls the function `_withdrawFromEscrow` in which it transfers NFT back from the escrow to the token holder.

```solidity
function _withdrawFromEscrow(
    CollectionType collectionType,
    address collection,
    address to,
    uint256 id
) internal returns (bool) {
    if (!_isEscrowed(collection, id)) {
        return false;
    }
    address from = address(this);
    if (collectionType == CollectionType.ERC721) {
        IERC721(collection).safeTransferFrom(from, to, id);
    } else {
        IERC1155(collection).safeTransferFrom(from, to, id, 1, "");
    }
    _escrow[collection][id] = address(0x0);
    return true;
}
```

If the token holder intentionally is a contract with no `onERC721Received` function selector, then the NFTS will be always stuck in the contract itself.

Remediation : Use `transferFrom` instead of `safeTransferFrom`.

> When there is an issue where tokens can become stuck or in the withdrawal mechanism, or where the operation should happen no matter what, it is advisable to use options like `transfer` and `transferFrom`. Otherwise, `safeTransfer` and `safeTransferFrom` are considered a good match.

Read more at : [1](https://solodit.cyfrin.io/issues/m-16-staked-service-will-be-irrecoverable-by-owner-if-not-an-erc721-receiver-code4rena-olas-olas-git) , [2](https://solodit.cyfrin.io/issues/tokens-irrecoverable-by-owner-on-l1-if-not-an-erc721-receiver-codehawks-arkproject-nft-bridge-git) , [3](https://solodit.cyfrin.io/issues/m-04-records-minted-to-an-address-that-is-a-smart-contract-that-cant-handle-erc721-tokens-will-be-stuck-forever-pashov-none-metalabel-markdown)

### 🚀 Use of `safeMint`

Example: In the [stationX](https://github.com/pashov/audits/blob/master/team/md/StationX-security-review.md) audit contest, which is a protocol for creating and managing DAOs, there was a reentrancy bug caused by the use of `safeMint`.

In the stationX codebase, there was a function called `buyGovernanceTokenERC721DAO` that allowed users to mint new DAOTokens as ERC721. The `buyGovernanceToken` function forwarded the call to `mintToken`, where the tokens were minted to the `_to` address using `safeMint`. Since the function did not have any reentrancy guard, a user could reenter if `_to` was a smart contract implementing a function selector like this:

`function onERC721Received(address, address, uint256, bytes memory) public virtual returns (bytes4)`

and in the function it called the stationX’s `buyGovernanceTokenERC721DAO` with the necessary parameters, this would have allowed the attacker or the \`to\` address to hold tokens more than `maxtokensPerUser` limit.

```solidity
    function mintToken(address _to, string calldata _tokenURI, uint256 _amount) public onlyFactory(factoryAddress) {
        if (balanceOf(_to) + _amount > erc721DaoDetails.maxTokensPerUser) {
            revert MaxTokensMintedForUser(_to);
        }

        if (!erc721DaoDetails.isNftTotalSupplyUnlimited) {
            require(
                Factory(factoryAddress).getDAOdetails(address(this)).distributionAmount >= _tokenIdTracker + _amount, "Max supply reached" );
        }
        for (uint256 i; i < _amount;) {
            _tokenIdTracker += 1;
            _safeMint(_to, _tokenIdTracker);
        }
```

Remediation: Use [ReentrancyGuard](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuard.sol) to prevent reentrancy attacks.

Read more at : [1](https://solodit.cyfrin.io/issues/h-01-bypassing-maxtokensperuser-limits-for-erc721-dao-pashov-audit-group-none-stationx-markdown)

### 🚀Use of `safetransfer` and `safetransferFrom`

Just like `safeMint`, both `safeTransferFrom` and `safeTransfer` check if the recipient can receive NFTs using the same function selector called `onERC721Received`. This can also lead to the issue of reentrancy.

Similar to the [Revert Lend Vault Contract](https://code4rena.com/reports/2024-03-revert-lend), which is a lending protocol specifically designed for liquidity providers on Uniswap v3 had a similar reentrancy issue.The vulnerability arises due to a potential reentrancy attack in the `onERC721Received` function in the [V3Vault](https://github.com/code-423n4/2024-03-revert-lend/blob/main/src/V3Vault.sol#L454-L473).sol, which handles the reception of an ERC721 token (a Uniswap V3 position token). Specifically, the issue is caused by the incorrect ordering of function calls that can lead to the manipulation of internal debt shares in the vault contract.

#### Problem:

1. When a position token is transformed, the contract checks if the token ID is different from the old one.
    
2. In the `else` block, the debt from the old token is copied to the new token (`loans[tokenId] = Loan(loans[oldTokenId].debtShares)`).
    
3. The contract then calls `_cleanupLoan()` to remove the old token and clear its loan data.
    
4. After `_cleanupLoan()`, `_updateAndCheckCollateral()` is called for the new token to update its collateral information.
    

However, if a malicious actor receives the old token back, they can trigger the `onERC721Received` callback again. This callback can modify the debt shares (via a `borrow` call), and since `_updateAndCheckCollateral()` is already called on the new token, a second invocation can lead to manipulated debt share values.

```solidity
function onERC721Received(address, address from, uint256 tokenId, bytes calldata data) 
    external override returns (bytes4) {
    ...
    if (tokenId != oldTokenId) {
        uint256 oldTokenId = transformedTokenId;
        address owner = tokenOwner[oldTokenId];

        // Copy debt to new token
        loans[tokenId] = Loan(loans[oldTokenId].debtShares);

        _addTokenToOwner(owner, tokenId);
        emit Add(tokenId, owner, oldTokenId);

        // Clears data of old loan
        _cleanupLoan(oldTokenId, debtExchangeRateX96, lendExchangeRateX96, owner);

        // Updates data of new loan
        _updateAndCheckCollateral(tokenId, debtExchangeRateX96, lendExchangeRateX96, 0, loans[tokenId].debtShares);
    }
    return IERC721Receiver.onERC721Received.selector;
}
```

```solidity
function _cleanupLoan(uint256 tokenId, uint256 debtExchangeRateX96, uint256 lendExchangeRateX96, address owner) internal {
    _removeTokenFromOwner(owner, tokenId);
    _updateAndCheckCollateral(tokenId, debtExchangeRateX96, lendExchangeRateX96, loans[tokenId].debtShares, 0);
    delete loans[tokenId];
    nonfungiblePositionManager.safeTransferFrom(address(this), owner, tokenId);
    emit Remove(tokenId, owner);
}
```

**Reentrancy Attack Scenario:**

1. When the vault receives the new token, `onERC721Received` is triggered, copying debt from the old token.
    
2. The old token is returned to the owner, triggering another `onERC721Received` call.
    
3. This callback can alter loan data and call `_updateAndCheckCollateral` again, causing incorrect debt share values and affecting internal token configurations.
    

Read more at : [1](https://solodit.cyfrin.io/issues/h-02-risk-of-reentrancy-onerc721received-function-to-manipulate-collateral-token-configs-shares-code4rena-revert-lend-revert-lend-git) [2](https://solodit.cyfrin.io/issues/h-06-owner-of-a-position-can-prevent-liquidation-due-to-the-onerc721received-callback-code4rena-revert-lend-revert-lend-git)

---

## 🚀Function signatures ?

A function signature is the unique identifier of a function within a smart contract. It includes the function name and its parameters.

For a function like this

```solidity
function transfer(address to, uint256 amount) public returns (bool)
```

Signature would be

```solidity
transfer(address,uint256)
```

when you make a function call to a contract, the function signature is used to encode the call data. This data helps the EVM identify which function is being called and with which arguments.

NFT Metadata function signatures

1. `baseURI` : The `baseURI` function returns a base Uniform Resource Identifier (URI) that serves as a prefix for all token-specific URIs.For example, if the base URI is `https://api.example.com/metadata/`, the complete URI for a token with ID `1` would be `https://api.example.com/metadata/1`.
    
2. `tokenURI` : The `tokenURI` function returns the complete URI for a specific token's metadata.
    

In the [arkproject](https://codehawks.cyfrin.io/c/2024-07-ark-project/results?lt=contest&sc=reward&sj=reward&page=1&t=report) contest, an NFT bridging protocol allows users to bridge between L1 and L2 (Starknet). There are escrows on both sides; if an NFT exists on one side, it is transferred to escrow, and a new custom NFT is minted for the user on the other side. To mint the same NFT on the other chain, we need to retrieve the NFT's metadata and then pass it to be minted on the other end.

There was a call to the `_callBaseUri` function in Arkproject’s tokenUtil contract, which helps retrieve metadata about the token.

```solidity
function _callBaseUri(
        address collection
    )
        internal
        view
        returns (bool, string memory)
{
    bool success;
    require(success == false);
    uint256 returnSize;
    uint256 returnValue;
    bytes memory ret;
    bytes[2] memory encodedSignatures = [abi.encodeWithSignature("_baseUri()"), abi.encodeWithSignature("baseUri()")]; 
    for (uint256 i = 0; i < 2; i++) {
        bytes memory encodedParams = encodedSignatures[i];
        assembly {
            success := staticcall(gas(), collection, add(encodedParams, 0x20), mload(encodedParams), 0x00, 0x20)
            if success {
                returnSize := returndatasize()
                returnValue := mload(0x00)
                ret := mload(0x40)
                mstore(ret, returnSize)
                returndatacopy(add(ret, 0x20), 0, returnSize)
                mstore(0x40, add(add(ret, 0x20), add(returnSize, 0x20)))
            }
        }
        if (success && returnSize >= 0x20 && returnValue > 0) {
            return (true, abi.decode(ret, (string)));
        }
    }
    return (false, "");
}
```

The issue arises from the `TokenUtil::_callBaseUri` function in the bridge contract. It attempts to get the `baseURI` of an NFT collection using incorrect function signatures. It calls `_baseUri()` and `baseUri()`, but the correct function signature in the ERC721 standard is `baseURI()`. Due to this error, the function always returns an empty string, even if the collection has a `baseURI` set.

**ERC721 Standard Function Signature:**

According to the OpenZeppelin documentation for ERC721, the function to get the base URI is `baseURI()`, which returns the base URI set using `_setBaseURI`.

> Make sure you pass all the correct parameters to the right function signatures. These small development mistakes can often lead to the failure of the entire protocol.

Read more at [1](https://codehawks.cyfrin.io/c/2024-07-ark-project/s/373) [2](https://solodit.cyfrin.io/issues/m-02-bytes-data-param-is-not-passed-to-erc721-recipient-as-expected-by-eip-721-code4rena-superposition-superposition-git) [3](https://solodit.cyfrin.io/issues/m-09-vault721tokenuri-does-not-comply-with-erc721-metadata-specification-code4rena-open-dollar-open-dollar-git)

## 🐋 Abusing approvals

The approval mechanism in the **ERC-721** standard enables the owner of a non-fungible token (NFT) to authorize another address to transfer or manage the token on their behalf.

When transferring an NFT from one user to another, it is essential to revoke the approvals associated with the previous owner. If these approvals are not revoked, there is a risk that the previous owner could exploit this oversight to steal NFTs or ERC20 tokens.

An issue was identified in the [Footium](https://github.com/sherlock-audit/2023-04-footium-judging) contest, where Footium, a football management game, allows players to own, manage, and develop virtual football clubs. **Player Minting and Transfers**: Players can scout and mint new talent from their club's youth academy as NFTs, facilitating the buying, selling, and trading of players on secondary markets.

The vulnerability stems from improper handling of **approvals** in the ERC-721 standard during ownership transfers. In Footium, a football management game, each club is represented by an NFT, and assets such as ERC-20 tokens are managed through a FootiumEscrow contract. The issue arises because approvals (granted using `setApprovalForERC20` and `setApprovalForERC721`) are not revoked when the club NFT is transferred to a new owner. This oversight allows a malicious previous owner to retain control over the escrowed assets, enabling them to misappropriate ERC-20 tokens or NFTs using the prior approvals. The root cause is the failure to reset or invalidate approvals upon NFT transfers. The impact is significant, as it permits unauthorized access to assets, potentially causing substantial losses for the new NFT owner. To address this, the protocol should revoke all existing approvals in the escrow when the club NFT is transferred. Alternatively, creating a new escrow for the new owner and transferring assets into it would ensure clear ownership..

```solidity
function setApprovalForERC20(
    IERC20 erc20Contract,
    address to,
    uint256 amount
) external onlyClubOwner {
    erc20Contract.approve(to, amount);
}

function setApprovalForERC721(
    IERC721 erc721Contract,
    address to,
    bool approved
) external onlyClubOwner {
    erc721Contract.setApprovalForAll(to, approved);
}
```

Read more at : [1](https://github.com/sherlock-audit/2023-04-footium-judging) [2](https://github.com/code-423n4/2024-08-superposition-findings/issues/56#event-14300401337)

## 🚀 Non-compliance with eip2981 (NFT royalty standard)

EIP-2981 lets NFT creators set a percentage of future sales to be paid to them as royalties.

```solidity
function royaltyInfo(uint256 tokenId, uint256 salePrice) external view returns (address receiver, uint256 royaltyAmount);
```

The proposal ensures that artists and creators receive compensation for their work beyond the initial sales, encouraging more creators to join NFT platforms. This was a standardization done because various NFT marketplaces were offering royalty services with their own implementations. The issue was that they could not share the NFT royalty info with other marketplaces if transferred to another marketplace. EIP-2981 solved that.

In execution for every token, there is a function called `royaltyInfo()`. This function takes a token ID and the sale price (the price it was sold for) and returns the address of the royalty receiver and the amount that must be transferred as a royalty.

```solidity
function royaltyInfo(uint256 _tokenId, uint256 _salePrice) external view returns (address receiver, uint256 royaltyAmount);
```

P.S.: These fees can be set by the admin or owner for each token ID by calling the internal function `_setTokenRoyalty(tokenId, receiver, feeNumerator)`, where `feeNumerator` represents the royalty percentage if you use the OpenZeppelin version of [ERC721 Royalty](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/extensions/ERC721Royalty.sol).

In the Flayer Protocol, a specialized NFT liquidity protocol that allows users to instantly convert their NFTs (both common and rare) into tradeable tokens called ƒ tokens, there was an issue where the marketplace itself was being set as the royalty holder.

Currently, both `ERC721Bridgable` and `ERC1155Bridgable` contracts implement royalties in a way that goes against this key principle. The issue lies in their `initialize` function:

```solidity
// Set this contract to receive marketplace royalty
_setDefaultRoyalty(address(this), _royaltyBps);
```

This implementation leads to two main problems:

1. It applies the same royalty rate to all NFTs in the collection.
    
2. It designates the bridge contract itself as the royalty recipient.
    

Consequently, all royalties are combined at a single, uniform rate, making it impossible to:

* Collect the correct royalty amounts for different pieces
    
* Attribute these royalties to their rightful creators
    

Read more at [1](https://solodit.cyfrin.io/issues/m-7-erc721bridgable-and-erc1155bridgable-are-not-eip-2981-compliant-and-fail-to-correctly-collect-or-attribute-royalties-to-artists-sherlock-flayer-git)

## Miscellaneous

## ⚡Off-by-one error

While performing operations like minting, burning, transferring, or checking an enforced limit, there can be an off-by-one error, where the upper or lower limit is not strictly enforced.

Example: In the [stationX](https://github.com/pashov/audits/blob/master/team/md/StationX-security-review.md) security review, it was found that while checking the maximum limit of NFT holdings for a user, there was an off-by-one error that allowed transferring one more NFT than intended.

```solidity
require(balanceOf(to) <= erc721DaoDetails.maxTokensPerUser);
```

This check lets the recipient address (`to`) hold up to the maximum limit (`maxTokensPerUser`). However, best practices suggest that the maximum limit should be strictly enforced, preventing values equal to the limit.

The solution implemented was to strictly enforce the limit, ensuring that users cannot exceed the maximum allowed.

```solidity
-   require(balanceOf(to) <= erc721DaoDetails.maxTokensPerUser);
+   require(balanceOf(to) < erc721DaoDetails.maxTokensPerUser);
```

Read more at : [1](https://solodit.cyfrin.io/issues/m-14-erc721-max-token-per-user-limit-can-always-be-bypassed-by-one-pashov-audit-group-none-stationx-markdown)

## ⚡Unnecessary pausing/blocking feature

In scenarios where it is critical for a user or administrator, **pausing functionality** is often employed as a precautionary measure to temporarily halt significant operations, typically during emergencies or contract upgrades. However, implementing pausing in situations where it is not absolutely necessary can lead to unforeseen issues, particularly in DeFi and NFT rental systems.

For instance, reNFT is a multi-chain NFT rental protocol and platform that provides collateral-free renting, lending, and reward-sharing features. In the reNFT contract, it managed Axie Infinity, an ERC721 with a pausing feature.

In [reNFT’s contract](https://github.com/re-nft/_v3_.smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Stop.sol#L353), the functions [stopRent and StopRentBatch](https://github.com/re-nft/_v3_.smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Stop.sol#L313) are utilized to cease NFT rentals by invoking an internal function `_reclaimRentedItems` to return tokens to the owner. However, if Axie Infinity's pausing feature is activated, it will not halt the rental process.

```solidity
function reclaimRentalOrder(RentalOrder calldata rentalOrder) external {
    if (address(this) == original) {
        revert Errors.ReclaimerPackage_OnlyDelegateCallAllowed();
    }
    if (address(this) != rentalOrder.rentalWallet) {
        revert Errors.ReclaimerPackage_OnlyRentalSafeAllowed(
            rentalOrder.rentalWallet
        );
    }
    uint256 itemCount = rentalOrder.items.length;
    for (uint256 i = 0; i < itemCount; ++i) {
        Item memory item = rentalOrder.items[i];
        // Transfering of nfts 
        if (item.itemType == ItemType.ERC721) item.transferERC721(rentalOrder.lender);
        if (item.itemType == ItemType.ERC1155)
            item.transferERC1155(rentalOrder.lender);
    }
}
```

Read more at : [1](https://solodit.cyfrin.io/issues/m-12-paused-erc721erc1155-could-cause-stoprent-to-revert-potentially-causing-issues-for-the-lender-code4rena-renft-renft-git) [2](https://codehawks.cyfrin.io/c/2024-07-ark-project/s/804)

> 💡Tips: *These pausing/whitelisting bugs are often found when assets are moved back from the protocol to the user. Pausing and whitelisting should not occur when withdrawal is necessary; otherwise, tokens can get stuck in the contract.*

## ⚡Whitelisting in cross-chain?

When bridging NFTs between Layer 1 (L1) and Layer 2 (L2), it's important to keep the NFT collections secure on both chains. A common security step is to whitelist NFT collections on the destination chain, allowing only trusted collections.

To ensure everything works correctly, it's essential to verify that whitelisting is effective in all relevant functions. Mistakes often occur with deposits and withdrawals, so both operations should be checked against whitelisting, and these checks must be done at all necessary steps.

In the context of cross-chain transfers, the checks to ensure are:

* Whitelisting should be verified on both chains (L1 and L2).
    
* Both withdrawing and depositing on both chains should confirm that whitelisting checks are in place.
    
* **Inability to Bridge from L1 to L2**: NFTs cannot be bridged from L1 to L2 if the collection is not whitelisted on L1.
    

One case study is the [arkproject](https://codehawks.cyfrin.io/c/2024-07-ark-project), where all necessary checks were missing, leading to mismatches in whitelisting across both chains. As a result, NFTs could be easily bridged from **L2 to L1**, but not the other way around. The protocol allowed users to bridge assets from L1 to L2 and vice versa.

The problem is with the whitelisting behavior on L1. Collections are only whitelisted when they are newly deployed.

```solidity
if (collectionL1 == address(0x0)) {
    if (ctype == CollectionType.ERC721) {
        collectionL1 = _deployERC721Bridgeable( ... );
        // Whitelist only when deploying a new collection
        _whiteListCollection(collectionL1, true); 
    } else {
        revert NotSupportedYetError();
    }
}
```

* **Bug**: If `collectionL1` already exists, it is not whitelisted, which causes bridging issues.
    

\*\*L2 Code (Bridge.cairo)\*\*On L2, collections are also whitelisted only during deployment:

```plaintext
if !collection_l2.is_zero() {
    // Existing collections are returned without being whitelisted
    return collection_l2;
}
// Whitelist only during new deployment
let (already_white_listed, _) = self.white_listed_list.read(l2_addr_from_deploy);
if already_white_listed != true {
    _white_list_collection(ref self, l2_addr_from_deploy, true);
}
```

* **Bug**: If `collection_l2` already exists, it is returned without whitelisting.
    

> Areas to focus on include: Bridging and cross-chain.

The solution was to whitelist the NFT regardless of its status.

Read more at [1](https://codehawks.cyfrin.io/c/2024-07-ark-project/s/672) [2](https://codehawks.cyfrin.io/c/2024-07-ark-project/s/548)

## ⚡ 1- way Transfer

In the ERC-20 standard, the `transferFrom` function allows token transfers between addresses, but only if there is prior approval. However, not all tokens strictly follow this standard. Some, like USDT, don't return a boolean value when executed, which can cause unexpected behaviors and potential security issues in smart contracts.

To address these inconsistencies and improve transaction safety, OpenZeppelin provides the `SafeERC20` library. This library includes wrapper functions, like `safeTransferFrom`, that ensure token transfers are secure by handling non-standard return values and reverting transactions if failures occur. By using `SafeERC20`, developers can protect their contracts from issues with non-compliant tokens, leading to more reliable token interactions within the Ethereum ecosystem.

Similarity between safeErc20’s `safetransferFrom` and erc721 ‘s `safetransferFrom`

NFT/ERC721

```solidity
    function safeTransferFrom(address from, address to, uint256 tokenId) public {
        safeTransferFrom(from, to, tokenId, "");
    }
```

ERC20

```solidity
    function safeTransferFrom(IERC20 token, address from, address to, uint256 value) internal {
        _callOptionalReturn(token, abi.encodeCall(token.transferFrom, (from, to, value)));
    }
```

The issue is that there are many similarities between the implementations of safeErc20 and erc721, which can lead to transferring NFTs using the safeErc20 implementation. This was observed in the [linea](https://github.com/solodit/solodit_content/blob/main/reports/Cyfrin/2024-05-24-cyfrin-linea.md) audit, where the bridgetoken function, intended to support only ERC20 tokens, was also able to transfer NFTs from L1 to L2 (linea). However, problems arose when attempting to bridge it back to the L1 side.

```solidity
  function bridgeToken(
    address _token,
    uint256 _amount,
    address _recipient
  ) public payable nonZeroAddress(_token) nonZeroAddress(_recipient) nonZeroAmount(_amount) whenNotPaused nonReentrant {
    uint256 sourceChainIdCache = sourceChainId;
    address nativeMappingValue = nativeToBridgedToken[sourceChainIdCache][_token];
    if (nativeMappingValue == RESERVED_STATUS) {
      revert ReservedToken(_token);
    }
    address nativeToken = bridgedToNativeToken[_token];
    uint256 chainId;
    bytes memory tokenMetadata;
    if (nativeToken != EMPTY) {
      BridgedToken(_token).burn(msg.sender, _amount);
      chainId = targetChainId;
    } else {
      uint256 balanceBefore = IERC20Upgradeable(_token).balanceOf(address(this));
      IERC20Upgradeable(_token).safeTransferFrom(msg.sender, address(this), _amount);
      _amount = IERC20Upgradeable(_token).balanceOf(address(this)) - balanceBefore;
      nativeToken = _token;
      if (nativeMappingValue == EMPTY) {
        nativeToBridgedToken[sourceChainIdCache][_token] = NATIVE_STATUS;
        emit NewToken(_token);
      }
      if (nativeMappingValue != DEPLOYED_STATUS) {
        tokenMetadata = abi.encode(_safeName(_token), _safeSymbol(_token), _safeDecimals(_token));
      }
      chainId = sourceChainIdCache;
    }
    messageService.sendMessage{ value: msg.value }(
      remoteSender,
      msg.value,
      abi.encodeCall(ITokenBridge.completeBridging, (nativeToken, _amount, _recipient, chainId, tokenMetadata))
    );
    emit BridgingInitiatedV2(msg.sender, _recipient, _token, _amount);
  }
```

Read more at [1](https://solodit.cyfrin.io/issues/tokenbridgebridgetoken-allows-1-way-erc721-bridging-causing-users-to-permanently-lose-their-nfts-cyfrin-none-cyfrin-linea-markdown)

---
