---
name: "fintech-patterns"
pack: "@rune/trading"
description: "Financial application patterns — safe money handling with Decimal/BigInt, transaction processing, audit trails, regulatory compliance, and PnL calculations."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# fintech-patterns

Financial application patterns — safe money handling with Decimal/BigInt, transaction processing, audit trails, regulatory compliance, and PnL calculations. Prevents the #1 fintech bug: float arithmetic on money.

#### Workflow

**Step 1 — Detect money handling code**
Use `Grep` to scan for raw float arithmetic on price/amount/balance fields: `Grep pattern="(price|amount|balance|pnl)\s*[\+\-\*\/]" glob="**/*.ts"`. Flag any result not wrapped in Decimal or BigInt.

**Step 2 — Enforce Decimal/BigInt boundaries**
Use `Read` on each flagged file to identify entry points (API response parsing, user input). Replace raw number literals with `new Decimal(value)` at parse time. All arithmetic must flow through Decimal operations until final display.

**Step 3 — Implement audit trail and verify rounding**
Use `Bash` to run `tsc --noEmit` confirming no implicit `any` on financial fields. Add an immutable audit log entry on every mutation (create, fill, cancel). Verify rounding mode is `ROUND_HALF_EVEN` (banker's rounding) for all display formatting.

#### Example

```typescript
import Decimal from 'decimal.js';

Decimal.set({ rounding: Decimal.ROUND_HALF_EVEN });

// NEVER: const fee = price * 0.001
// ALWAYS: Decimal arithmetic — exact, auditable
function calculateFee(price: string, quantity: string, feeRate: string): Decimal {
  return new Decimal(price)
    .times(new Decimal(quantity))
    .times(new Decimal(feeRate))
    .toDecimalPlaces(8);
}

function formatUSD(value: Decimal): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 2,
  }).format(value.toNumber());
}
```
