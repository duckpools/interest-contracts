# Interest Contract Documentation

## Overview

The Interest Contract serves as the interest rate oracle for the lending protocol. Its primary purpose is to **track and report the current value of borrow tokens**, which accumulates interest over time based on pool utilization. This enables the Pool and Collateral contracts to accurately calculate outstanding debt, loan amounts, and repayment requirements.

Any Interest Contract implementation must conform to the interface expected by the Pool and Collateral contracts. Beyond this interface, implementations are free to use any methodology for interest rate calculation and accrual.

---

## Purpose

The Interest Contract fulfills several critical functions:

1. **Borrow Token Valuation**: Reports the current value of borrow tokens in terms of pool currency
2. **Interest Accrual**: Accumulates interest over time, increasing the borrow token value
3. **Debt Calculation**: Enables consuming contracts to convert between borrow token amounts and actual debt owed
4. **Rate Discovery**: Calculates interest rates based on pool utilization (methodology is implementation-specific)

---

## Required Interface

**Every Interest Contract implementation MUST provide the following, as they are directly read by the Pool and Collateral contracts.**

### Token Identification — REQUIRED

The Interest Contract is identified by its first token (NFT). This NFT must be registered in the Pool and Collateral contracts at compile time:

```scala
val InterestNFT = fromBase58("{interestNft}")
```

### R5: Borrow Token Value (`BigInt`) — REQUIRED

A BigInt value representing the current value of borrow tokens:

| Register | Field | Description | Referenced By | Valid Range |
|----------|-------|-------------|---------------|-------------|
| R5 | `borrowTokenValue` | Current value of borrow tokens in pool currency (scaled by BorrowTokenDenomination) | Pool, Collateral | > 0 |

**Initial Value**: The borrow token value should be initialized to `BorrowTokenDenomination` (10,000,000,000,000,000) so that 1 borrow token equals 1 unit of pool currency at genesis.

**Value Growth**: This value increases over time as interest accrues. The rate of increase depends on the implementation's interest rate model.

---

## How Interest Contracts Are Located

### From Pool Contract

The pool locates the interest box by matching against the hardcoded Interest NFT:

```scala
val interestBox = CONTEXT.dataInputs.filter{
    (b : Box) => b.tokens.size > 0 && b.tokens(0)._1 == InterestNFT
}(0)
val borrowTokenValue = interestBox.R5[BigInt].get
```

### From Collateral Contract

The collateral contract uses the same mechanism:

```scala
val interestBox = CONTEXT.dataInputs.filter{
    (b: Box) => b.tokens.size > 0 && b.tokens(0)._1 == InterestNFT
}(0)
val borrowTokenValue = interestBox.R5[BigInt].get
```

---

## How R5 (borrowTokenValue) Is Used

### Constants Used by Consuming Contracts

| Constant | Value | Description |
|----------|-------|-------------|
| `BorrowTokenDenomination` | 10,000,000,000,000,000 (10^16) | Denominator for borrow token value calculations |

### Pool Contract Usage

During **borrow operations**, the pool uses `borrowTokenValue` to:

| Calculation | Formula | Purpose |
|-------------|---------|---------|
| Current borrowed amount | `currentBorrowTokensCirculating * borrowTokenValue / BorrowTokenDenomination` | Track total debt outstanding |
| Successor borrowed amount | `successorBorrowTokensCirculating * borrowTokenValue / BorrowTokenDenomination` | Validate debt changes |
| Loan amount | `collateralBorrowTokens._2 * borrowTokenValue / BorrowTokenDenomination` | Calculate actual loan value for threshold checks |
| Lend token value | `LendTokenMultiplier * (pooledAssets + borrowed) / lendTokensCirculating` | Determine LP token exchange rate |

### Collateral Contract Usage

The collateral contract uses `borrowTokenValue` in multiple operations:

**Debt Calculation (all operations)**:

```scala
val baseTotalOwed = loanAmount.toBigInt * borrowTokenValue / BorrowTokenDenomination
```

**Partial Repayment**:

| Calculation | Formula | Purpose |
|-------------|---------|---------|
| Expected borrow tokens after repayment | `currentBorrowTokens - (repaymentMade * BorrowTokenDenomination / borrowTokenValue)` | Validate correct token reduction |
| Final total owed | `finalBorrowTokens * borrowTokenValue / BorrowTokenDenomination` | Check remaining collateral sufficiency |

**Liquidation**:

| Calculation | Formula | Purpose |
|-------------|---------|---------|
| Total owed | `loanAmount * borrowTokenValue / BorrowTokenDenomination` | Determine if loan is underwater |
| Borrower share | `(quotePrice - totalOwed) * (PenaltyDenom - penalty) / PenaltyDenom` | Calculate liquidation proceeds distribution |

**Full Repayment**:

| Calculation | Formula | Purpose |
|-------------|---------|---------|
| Total owed | `loanAmount * borrowTokenValue / BorrowTokenDenomination` | Validate repayment amount exceeds debt |

---

## Conversion Formulas

### Borrow Tokens to Pool Currency

To convert borrow token amount to actual debt in pool currency:

```
debtAmount = borrowTokenAmount * borrowTokenValue / BorrowTokenDenomination
```

### Pool Currency to Borrow Tokens

To convert pool currency amount to borrow tokens:

```
borrowTokenAmount = currencyAmount * BorrowTokenDenomination / borrowTokenValue
```

---

## Security Requirements

Any Interest Contract implementation must ensure:

1. **Unique Interest NFT**: The Interest NFT must have exactly 1 token minted, and the interest contract box must hold this NFT. This ensures the borrow token value cannot be forged by creating duplicate NFTs.

2. **Monotonically Increasing Value**: The `borrowTokenValue` should only increase over time (or remain constant). Decreasing values would allow borrowers to pay back less than they owe.

3. **Value Bounds**: The `borrowTokenValue` must remain positive and within bounds that prevent overflow when multiplied with borrow token amounts in consuming contracts.

4. **Data Input Usage**: The interest box is used as a data input (read-only) by Pool and Collateral contracts, meaning its state is not modified during those transactions.

---

## Implementation Checklist

When creating a new Interest Contract:

- [ ] Box has unique identifying NFT as `tokens(0)`
- [ ] NFT identifier is compiled into Pool and Collateral contracts
- [ ] R5 contains a BigInt value representing `borrowTokenValue`
- [ ] R5 initial value equals `BorrowTokenDenomination` (10^16)
- [ ] R5 value only increases over time (interest accrual)
- [ ] Interest rate calculation methodology is defined
- [ ] Update mechanism ensures timely rate adjustments
- [ ] Value bounds prevent overflow in consuming contract calculations
- [ ] Box can be used as a data input by Pool and Collateral contracts
