Here’s the refined and formatted version of the notes, with consistent structure and improved clarity:

---

### ERC-4626 Overview

- ERC-4626 is an extension of the ERC-20 standard.
- It introduces vault functionality, enabling users to deposit tokens and receive shares that represent their ownership of the vault's assets.
- The number of shares issued corresponds to the proportion of assets deposited.

```solidity
abstract contract ERC4626 is ERC20 {}
```

---

### Functions

#### **`deposit()`**

Deposits a specified amount of tokens into the vault and mints shares for the user.

**Key Parameters:**

- `assets`: The number of tokens to deposit.
- `receiver`: The address that will receive the minted shares.

**Flow:**

1. Calculate the number of shares using `previewDeposit(assets)`.
2. Transfer the specified `assets` to the vault.
3. Mint the calculated `shares` to the `receiver`.
4. Emit the `Deposit` event.
5. Call `afterDeposit` (internal hook) for additional post-deposit logic.

```solidity
function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {
    // Ensure no rounding errors in shares calculation
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    // Transfer assets to the vault
    asset.safeTransferFrom(msg.sender, address(this), assets);

    // Mint the shares to the receiver
    _mint(receiver, shares);

    // Emit the Deposit event
    emit Deposit(msg.sender, receiver, assets, shares);

    // Post-deposit logic
    afterDeposit(assets, shares);
}
```

---

#### **`mint()`**

Mints a specified number of shares for the user by calculating the required assets.

**Key Parameters:**

- `shares`: The number of shares the user wants to mint.
- `receiver`: The address that will receive the minted shares.

**Flow:**

1. Calculate the required `assets` using `previewMint(shares)`.
2. Transfer the calculated `assets` to the vault.
3. Mint the requested `shares` to the `receiver`.
4. Emit the `Deposit` event.
5. Call `afterDeposit` for post-mint actions.

```solidity
function mint(uint256 shares, address receiver) public virtual returns (uint256 assets) {
    // Calculate required assets
    assets = previewMint(shares);

    // Transfer assets to the vault
    asset.safeTransferFrom(msg.sender, address(this), assets);

    // Mint the shares to the receiver
    _mint(receiver, shares);

    // Emit the Deposit event
    emit Deposit(msg.sender, receiver, assets, shares);

    // Post-mint logic
    afterDeposit(assets, shares);
}
```

---

### Key Difference: `deposit()` vs. `mint()`

- **`deposit()`**: Users specify the amount of tokens (`assets`) they want to deposit. The vault calculates and mints the corresponding shares.
- **`mint()`**: Users specify the number of shares they want. The vault calculates and collects the corresponding tokens (`assets`).

---

#### **`withdraw()`**

Allows users to withdraw a specific amount of assets by burning shares.

**Key Parameters:**

- `assets`: The amount of tokens to withdraw.
- `receiver`: The address to receive the withdrawn tokens.
- `owner`: The address whose shares will be burned.

**Flow:**

1. Calculate the required `shares` using `previewWithdraw(assets)`.
2. Verify authorization to burn shares (if the caller is not the `owner`).
3. Burn the `shares` from the `owner`.
4. Transfer the `assets` to the `receiver`.
5. Emit the `Withdraw` event.
6. Perform `beforeWithdraw` and `afterWithdraw` logic.

```solidity
function withdraw(
    uint256 assets,
    address receiver,
    address owner
) public virtual returns (uint256 shares) {
    // Calculate required shares
    shares = previewWithdraw(assets);
    require(shares != 0, "ZERO_SHARES");

    // Verify authorization
    if (msg.sender != owner) {
        uint256 allowed = allowance(owner, msg.sender);
        require(allowed >= shares, "ALLOWANCE_EXCEEDED");
        _approve(owner, msg.sender, allowed - shares);
    }

    // Perform pre-withdrawal actions
    beforeWithdraw(assets, shares);

    // Burn shares
    _burn(owner, shares);

    // Transfer assets to the receiver
    asset.safeTransfer(receiver, assets);

    // Emit the Withdraw event
    emit Withdraw(msg.sender, receiver, owner, assets, shares);

    // Post-withdrawal actions
    afterWithdraw(assets, shares);
}
```

---

#### **`redeem()`**

Burns a specific number of shares to withdraw the corresponding amount of assets.

**Key Parameters:**

- `shares`: The number of shares to redeem.
- `receiver`: The address to receive the withdrawn tokens.
- `owner`: The address whose shares will be burned.

**Flow:**

1. Calculate the assets using `previewRedeem(shares)`.
2. Verify authorization to burn shares (if the caller is not the `owner`).
3. Burn the `shares` from the `owner`.
4. Transfer the `assets` to the `receiver`.
5. Emit the `Withdraw` event.
6. Perform `beforeWithdraw` and `afterWithdraw` logic.

```solidity
function redeem(
    uint256 shares,
    address receiver,
    address owner
) public virtual returns (uint256 assets) {
    // Calculate corresponding assets
    assets = previewRedeem(shares);
    require(assets != 0, "ZERO_ASSETS");

    // Verify authorization
    if (msg.sender != owner) {
        uint256 allowed = allowance(owner, msg.sender);
        require(allowed >= shares, "ALLOWANCE_EXCEEDED");
        _approve(owner, msg.sender, allowed - shares);
    }

    // Perform pre-redemption actions
    beforeWithdraw(assets, shares);

    // Burn shares
    _burn(owner, shares);

    // Transfer assets to the receiver
    asset.safeTransfer(receiver, assets);

    // Emit the Withdraw event
    emit Withdraw(msg.sender, receiver, owner, assets, shares);

    // Post-redemption actions
    afterWithdraw(assets, shares);
}
```

---

### Share Allocation Formula

To maintain equity:

If a deposit increases the vault’s token balance by a certain percentage, the total shares must increase proportionally.

Let a =amount to deposit 
let B= total shares
let s = shares to allocate
let T=total shares 


As the deposit increases the number of tokens by % certain %, then the total shares must increase by the same %.


Total increase in assets = (a +B ) * 100 / B

Total increase in shares = (s + T) * 100 / T

Now as % should be same so 
![[Pasted image 20250114123540.png]]



On withdrawing the assets, the decrease in % should be same.

B= Total Tokens 
a= Amount to withdraw 
s=shares to burn 
T= total shares 
![[Pasted image 20250114125119.png]]



**Formula:**  
Shares to allocate = (Amount to deposit × Total shares) ÷ Total assets

assets to shares = (shares × Total assets) ÷ Total shares
![[Pasted image 20250114125312.png]]

---

#### **Conversion Functions**

1. **To Shares:**  
    Converts a specified amount of assets into shares using the above formula.
    
    ```solidity
    function convertToShares(uint256 assets) public view returns (uint256 shares) {
        shares = (assets * totalSupply()) / totalAssets();
    }
    ```
    
2. **To Assets:**  
    Converts a specified number of shares into assets.
    
    ```solidity
    function convertToAssets(uint256 shares) public view returns (uint256 assets) {
        assets = (shares * totalAssets()) / totalSupply();
    }
    ```