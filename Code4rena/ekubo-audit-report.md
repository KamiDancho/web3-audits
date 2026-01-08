# 🧾 Ekubo Audit Findings

This document contains my findings from the **Ekubo** audit contest.  
Each issue includes context, impact, and proposed mitigations.

**Platform:** Code4rena  
**Contest:** Ekubo  
**Dates:** Nov 19, 2025 – Dec 10, 2025  
**Role:** Warden  
**Findings:** 4 Lows  

---

# [L-01] `withdrawAndRoll` Permanently Retains 1 Wei of Each Token

**Contract Name:** `PositionsOwner.sol`  
**Function Name:** `withdrawAndRoll`

---

### 🧠 Description
The `withdrawAndRoll()` function intentionally reduces each withdrawal amount by 1 wei using the pattern `sub(amount, gt(amount, 0))`.  
This design choice is intended to keep storage slots warm and reduce gas costs across repeated calls.

As a consequence, **1 wei of each token always remains inside the Positions contract** and cannot be withdrawn through this function.  
Over time, this leads to a permanent dust balance and a small but persistent divergence between actual token balances and the protocol’s internal accounting, since a “full” withdrawal never truly clears all fees.

---

### 🧰 Mitigation
Either explicitly accept and document this behavior as a gas-optimization trade-off, or provide a dedicated administrative function to recover residual dust:

~~~solidity
function withdrawDust(
    address token0,
    address token1,
    address recipient
) external onlyOwner {
    (uint128 amount0, uint128 amount1) =
        POSITIONS.getProtocolFees(token0, token1);

    if (amount0 > 0 || amount1 > 0) {
        POSITIONS.withdrawProtocolFees(
            token0,
            token1,
            amount0,
            amount1,
            recipient
        );
    }
}
~~~

---

### 💥 Impact
- A minimum of 1 wei per token is permanently retained  
- Protocol fee balances can never be fully cleared via standard withdrawal  
- Economically negligible, but introduces a long-lived accounting inconsistency  

---

# [L-02] `forward()` Allows Calldata Underflow Leading to Out-of-Gas Revert

**Contract Name:** `FlashAccountant.sol`  
**Function Name:** `forward`

---

### 🧠 Description
The `forward()` function performs a calldata copy using `sub(calldatasize(), 36)` without verifying that the calldata length is at least 36 bytes.

If the function is invoked with calldata shorter than 36 bytes, this subtraction underflows to `type(uint256).max`.  
The subsequent `calldatacopy` then attempts to copy an excessively large amount of data, inevitably resulting in an out-of-gas revert.

---

### 🧰 Mitigation
Introduce a minimum calldata length check before executing the assembly logic:

~~~solidity
function forward(address to) external {
    Locker locker = _requireLocker();

    if (msg.data.length < 36) {
        revert InvalidCalldataLength();
    }

    assembly ("memory-safe") {
        // existing logic
        calldatacopy(add(free, 36), 36, sub(calldatasize(), 36))
    }
}
~~~

---

### 💥 Impact
- Callers can unintentionally trigger out-of-gas reverts  
- Gas is wasted for the caller only  
- No funds are at risk and no protocol-wide denial of service occurs  

---

# [L-03] Zero-Liquidity Deposits Are Not Explicitly Rejected

**Contract Name:** `BasePositions.sol`  
**Function Name:** `deposit`

---

### 🧠 Description
The `deposit()` function computes the maximum possible liquidity and enforces a `minLiquidity` constraint.  
However, it does not explicitly prevent execution when the calculated liquidity is zero.

If a user sets `minLiquidity = 0`, the function can proceed even when `liquidity == 0`, resulting in a no-op deposit that still consumes gas and executes unnecessary logic.

---

### 🧰 Mitigation
Add an explicit guard to prevent zero-liquidity deposits:

~~~solidity
liquidity = maxLiquidity(...);

if (liquidity == 0) {
    revert ZeroLiquidityDeposit();
}

if (liquidity < minLiquidity) {
    revert DepositFailedDueToSlippage(liquidity, minLiquidity);
}
~~~

---

### 💥 Impact
- Users can waste gas on ineffective deposits  
- No loss of funds  
- Degraded user experience due to unnecessary transactions  

---

# [L-04] Fees Are Silently Dropped When Pool Liquidity Is Zero

**Contract Name:** `Core.sol`  
**Function Name:** `accumulateAsFees`

---

### 🧠 Description
The `accumulateAsFees()` function allows fee accumulation even when a pool’s liquidity is zero.  
In this case, the function skips updating the pool’s fees-per-liquidity, effectively discarding the provided fee amounts.

This behavior does not revert and does not clearly signal to callers that fees were ignored.  
Extensions that call this function before any liquidity is added may unknowingly lose fees, even though the call appears to succeed.

---

### 🧰 Mitigation
Consider reverting when fees are provided while liquidity is zero, buffering the fees for later application, or emitting a dedicated event indicating that fee accumulation was skipped:

~~~solidity
if (liquidity == 0 && (amount0 != 0 || amount1 != 0)) {
    revert ZeroLiquidityPool();
}
~~~

---

### 💥 Impact
- Fees can be permanently lost if accumulated before liquidity is added  
- Silent behavior may surprise integrators  
- No direct security risk, but leads to unexpected economic outcomes  
