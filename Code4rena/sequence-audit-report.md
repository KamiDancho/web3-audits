# 🧾 Sequence Audit Findings

This document contains my findings from the **Sequence: Transaction Rails** audit contest on Code4rena.  
Each issue includes context, impact, and proposed mitigations.

**Platform:** Code4rena  
**Contest:** Sequence  
**Dates:** Nov 11, 2025 - Nov 17, 2025  
**Role:** Warden  
**Findings:** 2 Lows

## Summary Table

| ID | Title | Severity |
|----|-------|----------|
| Low-01 | Delegatecall to External Multicall3 Without Verification Weakens Router Trust Boundary | Low |
| Low-02 | Fee Commitments Can Be Bypassed by Setting `feeCollector = address(0)` | Low |


---

---

# [L-01] Delegatecall to external Multicall3 without verification weakens router trust boundary

**Contract Name:** `TrailsRouter.sol`  
**Functions:** `execute()`, `pullAmountAndExecute()`

---

### 🧠 Description
The `TrailsRouter` contract relies on a hardcoded third-party address referred to as `MULTICALL3`.  
During intent execution, the router pulls user funds into its own balance and then performs a `delegatecall` to this external contract in order to invoke the `aggregate3Value` logic.

The implementation presumes that `MULTICALL3` always contains the canonical and stateless Multicall3 bytecode.  
However, no on-chain mechanism confirms that this assumption is valid. If the router is deployed on a chain where the address does not host the expected contract—or if that address is replaced with a different implementation—the delegated call would run arbitrary code in the context of the router.

Since `delegatecall` preserves the caller’s storage and balances, such a scenario could allow unintended manipulation of ETH or ERC-20 tokens held by the router, breaking the intended security boundary.

---

### 💥 Impact
The router’s safety depends entirely on correct deployment configuration.  
A misconfigured or compromised `MULTICALL3` address may lead to uncontrolled code execution over assets managed by the router, undermining the protocol’s trust model.

---

### 🧰 Recommendation
Introduce explicit validation of the external contract before any delegated execution.  
The router should check that the code at `MULTICALL3` matches an expected and preapproved hash value for the specific chain.

Example mitigation logic:

```solidity
bytes32 expectedHash = 0x...; // canonical Multicall3 hash for the target chain
if (MULTICALL3.codehash != expectedHash) revert Errors.E_InvalidMulticall3();
```

As an additional improvement, consider making the `MULTICALL3` address immutable and configurable at deployment with a one-time on-chain attestation.  
For stronger isolation, the protocol could inline a minimal implementation of `aggregate3Value` or use an internal trusted library instead of delegating to external code.

---

---

# [L-02] Fee commitments can be bypassed by setting `feeCollector = address(0)`

**Contract Name:** `TrailsRouter.sol`  
**Functions:** `depositToIntent()`, `depositToIntentWithPermit()`

---

### 🧠 Description
The router supports intents that include optional relayer fees defined by the parameters `feeAmount` and `feeCollector`.

Both `depositToIntent()` and `depositToIntentWithPermit()` allow a user to sign an intent specifying a positive `feeAmount`.  
Nevertheless, the fee transfer is executed only when the collector address is different from zero. If `feeCollector` is set to `address(0)`, the functions silently skip the payment while processing the remainder of the deposit.

Attack path: a relayer encourages a user to approve an intent with `feeAmount > 0`, yet submits it for execution with `feeCollector = address(0)`.  
The transaction succeeds, no fee is transferred, and no event signals that the commitment was ignored. Off-chain reviewers examining the signed payload may assume the specified fee was honored, even though on-chain enforcement deliberately bypassed it.

This creates an inconsistency between off-chain fee expectations and actual contract behavior.

---

### 💥 Impact
Dishonest relayers can execute signed intents without paying the declared fee, obtaining services at no cost.  
Users or downstream systems relying on signed parameters may believe that a fee was settled when it was silently omitted.

---

### 🧰 Recommendation
Ensure that fees cannot be skipped without explicit acknowledgment.

The contract should revert whenever a positive fee is specified but the collector address is invalid.

Example adjustment:

```solidity
if (feeAmount > 0 && feeCollector == address(0)) {
    revert Errors.E_InvalidFeeCollector();
}
```

Alternatively, enforce `feeCollector != address(0)` as part of the intent signature schema so that off-chain commitments always match on-chain execution rules.


