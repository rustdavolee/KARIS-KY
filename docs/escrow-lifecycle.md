# Escrow Lifecycle — State Machine Reference

This document describes the `InvoiceEscrow.status` state machine, valid transitions,
forbidden regressions, and interaction rules between `withdraw` vs `settle` paths.

---

## Status values

| Value | Name | Meaning |
|-------|------|---------|
| `0` | `open` | Escrow is initialized; funding is active |
| `1` | `funded` | At least one investor reached or exceeded the funding target |
| `2` | `settled` | SME has finalized settlement after legal/financial review |
| `3` | `withdrawn` | SME has withdrawn liquidity (pull model, off-chain settlement) |
| `4` | `cancelled` | Admin cancelled the escrow before it was funded; investors may reclaim principal via `refund()` |
| `5` | `archived` | Admin has archived the escrow; read-only terminal state for closed records |

---

## State diagram

```text
                ┌─────────────┐
                │   (init)    │
                │  status = 0 │
                │    open      │
                └──────┬──────┘
                       │
         ┌─────────────┼──────────────────────┐
         │             │                      │
         │ fund(amount >= funding_target)      │ cancel_funding() [admin]
         ▼             │                      ▼
  ┌─────────────┐      │               ┌─────────────┐
  │  funded     │      │               │  cancelled  │
  │ status = 1  │      │               │  status = 4 │
  └──────┬──────┘      │               └──────┬──────┘
         │             │                      │
  ┌──────┼──────┐      │ (more funding        │ refund(investor) [investor]
  │      │      │      │  if target not met)  │ → returns InvestorContribution
  ▼      ▼      │      │                      ▼
┌────┐ ┌────┐   │      │               (principal returned)
│ 2  │ │ 3  │   └──────┘                     │
│set │ │wd  │                                 │
└──┬─┘ └──┬─┘                                 │
   │      │                                   │
   │      └──────────┬────────────────────────┘
   │                 │
   ▼                 ▼
┌─────────────────────────────┐
│  archive_escrow() [admin]   │
│        status = 5           │
│        archived             │
└─────────────────────────────┘
```

---

## Batch funding (`batch_fund`)

`batch_fund(contributions: Vec<(Address, i128)>)` processes multiple investor contributions in a single call,
reducing transaction overhead for primary issuance workflows.

**Semantics:**
- Each entry `(investor_address, amount)` is processed sequentially
- Per-investor `require_auth()` is called for each entry
- All existing [`fund()`](../escrow/src/lib.rs#L1444) invariants (allowlist, caps, min contribution, overflow guards)
  are enforced per entry
- One `EscrowFunded` event is emitted per entry
- One `BatchFundCompleted` summary event is emitted after all entries succeed
- If any entry fails its invariants, the call returns an error **without corrupting prior entries**
  (Soroban's transaction atomicity ensures consistent state)

**Capacity:**
- Batch size must be `> 0` and `<= MAX_FUND_BATCH` (50 entries)
- Empty batch panics with `EscrowError::FundingBatchEmpty`
- Oversized batch panics with `EscrowError::FundingBatchTooLarge`

**Funded-target snapshot:**
- If any entry causes the escrow to transition to **funded** (status `0 → 1`),
  `FundingCloseSnapshot` is recorded exactly once
- Remaining entries are processed even after the transition

**Example:**
```rust
let entries = vec![
    (investor_a, 30_000i128),
    (investor_b, 55_000i128),
    (investor_c, 10_000i128),
];
let result = batch_fund(entries); // All three funded in one call
```

---

## Valid transitions

| From | To | Trigger | Auth required |
|------|----|---------|--------------|
| `0` (open) | `1` (funded) | `fund()`, `fund_with_commitment()`, or `batch_fund()` when `funded_amount >= funding_target` | Investor auth (per-investor for batch) |
| `0` (open) | `4` (cancelled) | `cancel_funding()` | Admin auth; legal hold must be inactive |
| `1` (funded) | `2` (settled) | `settle()` | SME auth; legal hold must be inactive; if `maturity > 0`, ledger timestamp must be >= maturity |
| `1` (funded) | `3` (withdrawn) | `withdraw()` | SME auth; legal hold must be inactive |
| `2` (settled) | `5` (archived) | `archive_escrow()` | Admin auth |
| `3` (withdrawn) | `5` (archived) | `archive_escrow()` | Admin auth |
| `4` (cancelled) | `5` (archived) | `archive_escrow()` | Admin auth |

---

## Forbidden transitions (must panic)

| From | To | Reason |
|------|----|--------|
| `0` (open) | `1` (funded) | Must reach funding target first |
| `0` (open) | `2` (settled) | Escrow must be funded first |
| `0` (open) | `3` (withdrawn) | Escrow must be funded first |
| `1` (funded) | `0` (open) | Status never regresses |
| `1` (funded) | `4` (cancelled) | `cancel_funding` only allowed in Open state |
| `2` (settled) | any | Status never regresses from terminal |
| `3` (withdrawn) | any | Status never regresses from terminal |
| `4` (cancelled) | any | Status never regresses from terminal |
| `5` (archived) | any | Status never regresses from archived |

---

## Mutual exclusivity: `withdraw` vs `settle`

`withdraw` and `settle` are **mutually exclusive** terminal paths. Both require:
- `status == 1` (funded)
- No active legal hold
- SME authentication

Once one path is taken, the other is unreachable:
- After `withdraw()` → status is `3`; `settle()` panics
- After `settle()` → status is `2`; `withdraw()` panics

---

## Partial settlement (`partial_settle` / `settle(Some(amount))`)

Full settlement pays investors the entire outstanding principal plus yield in one shot.
**Partial settlement** lets the SME release a portion of the outstanding principal now and
settle the remainder later, without leaving the `funded` state. This is useful when the SME
receives cash in tranches (e.g. a customer pays an invoice in installments) and wants to
pass each tranche through to investors as it arrives.

### How it works

- `partial_settle(amount)` (equivalently `settle(Some(amount))`) requires `status == 1` (funded),
  SME auth, and no active legal hold — the same gates as full `settle()`.
- `amount` must be `> 0` and `<= outstanding principal`. Settling the full outstanding principal
  via `partial_settle` is equivalent to a full `settle()` and transitions `status 1 → 2`.
- Each partial settlement **distributes `amount` of principal pro-rata** across investors based on
  their contribution share, and records the cumulative amount distributed so far.
- The escrow **stays in `funded`** after a partial settlement; investors can claim their share
  immediately via `claim_investor_payout()`.
- A later `settle()` (or `partial_settle` of the remaining balance) finalizes the escrow.

### Investor-facing effect

Investors are paid **incrementally, as each tranche is settled**, rather than waiting for a single
final settlement. Each investor's claimable amount after a partial settlement is:

```
claimable_i = amount * (contribution_i / total_principal) - already_claimed_i
```

Because distribution is pro-rata, every investor receives the same fraction of their contribution
at each step. Investors who have not yet claimed simply accumulate a larger claimable balance.

### Worked numeric example

Assume an escrow with `funding_target = 100_000` and three investors:

| Investor | Contribution | Share |
|----------|--------------|-------|
| Alice | 50_000 | 50% |
| Bob | 30_000 | 30% |
| Carol | 20_000 | 20% |
| **Total** | **100_000** | **100%** |

**Step 1 — SME calls `partial_settle(40_000)`.**

40% of principal is distributed pro-rata:

| Investor | Distributed this step | Cumulative distributed |
|----------|-----------------------|------------------------|
| Alice | 40_000 × 50% = 20_000 | 20_000 |
| Bob | 40_000 × 30% = 12_000 | 12_000 |
| Carol | 40_000 × 20% = 8_000 | 8_000 |
| **Total** | **40_000** | **40_000** |

Status remains `1` (funded). `get_distributed_principal()` now returns `40_000`.

**Step 2 — SME calls `partial_settle(60_000)` to settle the remainder.**

| Investor | Distributed this step | Cumulative distributed |
|----------|-----------------------|------------------------|
| Alice | 60_000 × 50% = 30_000 | 50_000 |
| Bob | 60_000 × 30% = 18_000 | 30_000 |
| Carol | 60_000 × 20% = 12_000 | 20_000 |
| **Total** | **60_000** | **100_000** |

Because the full outstanding principal has now been distributed, the escrow transitions
`status 1 → 2` (settled). `get_distributed_principal()` returns `100_000`.

### Partial settlement invariants

The partial settlement feature provides **incremental pro-rata distribution** of principal to
investors. To ensure correctness and prevent mis-accounting, the following invariants are defined
and enforced:

#### Invariant 1: `get_distributed_principal()` advances monotonically

`get_distributed_principal()` returns the **cumulative principal that has been distributed to
investors so far** across all settlements (partial and final). It is the running total of every
`partial_settle`/`settle` amount applied to principal.

- Before any settlement: `0`.
- After a partial settlement of `amount`: increases by exactly `amount`.
- After full settlement: equals the total funded principal.
- **Invariant:** The value is monotonically non-decreasing. Once set to `X`, it can never decrease.
- **Invariant:** Attempting to settle an amount such that `distributed_principal + amount > funded_amount` panics.

#### Invariant 2: Investor payouts across multiple partial settle + claim sequences reconcile correctly

Each investor receives a **pro-rata share** of each settlement, independent of when they claim:

```
payout_i(step) = settled_amount(step) × (contribution_i / total_principal) + accrued_yield(step)
```

The **sum of all incremental payouts** (across all claim calls following multiple partial settlements)
equals the payout from a single full settlement at the same yield:

```
sum(payout_i(step1), payout_i(step2), ..., payout_i(stepN)) = payout_i(full_settlement)
```

This holds because:
- Each partial settlement distributes a fraction of the total principal.
- Yield is computed on the **settled amount** only, not on the full funded amount.
- The cumulative settled amount eventually equals the full funded amount.
- The cumulative yield from all partial settlements equals the yield from one full settlement.

#### Invariant 3: Partial claim then full settlement yields no double-counting

When an investor claims a payout after a partial settlement, then another partial settlement occurs,
the second payout includes only the **new incremental principal and yield**, not a re-distribution
of the already-claimed portion. This is achieved by marking each investor as claimed after their
first claim, ensuring idempotent behavior.

#### Invariant 4: `claim_investor_payout()` is idempotent across all settlement states

Once an investor has claimed after a settlement step, a second call to `claim_investor_payout()`
is a **no-op** and does not emit a duplicate event. The investor's claimed flag is set to `true`
after the first claim, and subsequent calls return early without further state mutations.

#### Invariant 5: Multiple investors with equal contributions receive equal payouts

For any settlement step, two investors with identical contributions receive identical payout
amounts, subject to rounding differences in integer division. This ensures **equitable distribution**:

```
contribution_a == contribution_b  ⟹  payout_a ≈ payout_b  (within rounding tolerance)
```

#### Invariant 6: Settling beyond outstanding principal panics

The sum of all partial settlements must not exceed the total funded principal:

```
sum(partial_amounts) ≤ funded_amount
```

Attempting to settle an amount that would exceed this invariant panics with
`EscrowError::SettlementExceedsFunded`.

### `get_distributed_principal`

`get_distributed_principal()` is the runtime accessor for Invariant 1. It returns the
**cumulative principal that has been distributed to investors so far** across all settlements
(partial and final).

It lets SMEs and investors verify how much principal has been released without reconstructing
the history of settlement calls, and it is the value used to compute the remaining outstanding
principal (`total_principal - get_distributed_principal()`) that a subsequent `partial_settle`
may distribute.

### Test coverage for partial settlement invariants

The invariants are verified by comprehensive test cases in
`escrow/src/tests/partial_settle_invariants.rs`:

- **Invariant 1:** `distributed_principal_advances_through_partial_settlements()`
- **Invariant 2:** `single_investor_partial_then_full_settlement_yields_match()`,
  `equal_investors_receive_equal_pro_rata_shares()`,
  `unequal_investors_receive_proportional_pro_rata_shares()`,
  `full_reconciliation_multi_partial_then_full()`,
  `multi_investor_full_reconciliation_partial_then_full()`
- **Invariant 4:** `double_claim_is_idempotent()`,
  `investor_accumulates_claims_across_partial_settlements()`
- **Invariant 6:** `settle_beyond_outstanding_principal_panics()`

Each test sets up a fresh escrow, executes one or more partial settlements, investor claims,
and verifies the payout invariants hold end-to-end.

---

## Investor refund path (status 4 — cancelled)

When an escrow is cancelled before reaching its funding target, investors may recover
their principal:

1. Admin calls `cancel_funding()` — transitions `status 0 → 4`. Blocked by legal hold.
2. Each investor calls `refund(investor)` — transfers exactly `DataKey::InvestorContribution`
   back to the investor via `external_calls::transfer_funding_token_with_balance_checks`.
3. `InvestorContribution` is zeroed after transfer (checks-effects-interactions pattern).
4. `DataKey::InvestorRefunded` is set to `true` — `is_investor_refunded()` returns `true`.
5. A second `refund()` call panics with `"no contribution to refund"` (contribution is 0).

### Invariants

- Total refunded ≤ `funded_amount` (each investor can only reclaim their own contribution).
- No double-refund: contribution is zeroed before the token transfer.
- Balance-delta checks enforced by `external_calls` wrapper (SEP-41 conservation).
- `refund()` is blocked in all states except `4` (cancelled).

### Events emitted

| Event | When |
|-------|------|
| `FundingCancelled` | `cancel_funding()` succeeds |
| `InvestorRefundedEvt` | `refund()` succeeds |

---

## SME auth vs admin role

| Function | Role |
|----------|------|
| `settle()` | SME |
| `withdraw()` | SME |
| `cancel_funding()` | Admin only |
| `set_legal_hold()` | Admin only |
| `update_maturity()` | Admin only (extensions require timelock/guardian; see below) |
| `propose_maturity_extension()` | Admin only |
| `confirm_maturity_extension()` | Admin only (after delay or guardian confirmation) |
| `propose_admin()` | Admin only |
| `accept_admin()` | Pending admin only |
| `propose_upgrade()` | Admin only |
| `execute_upgrade()` | Admin only (after delay) |

The SME role represents the off-chain settlement policy authority. The admin role
handles on-chain configuration and compliance controls.

---

## Legal hold interaction

Legal hold blocks all risk-bearing operations regardless of status:

| Function | Blocked by legal hold |
|----------|----------------------|
| `fund()` | Yes |
| `settle()` | Yes |
| `withdraw()` | Yes |
| `claim_investor_payout()` | Yes |
| `cancel_funding()` | Yes |
| `sweep_terminal_dust()` | Yes |

Once legal hold is cleared, normal state transitions resume.

---

## Maturity gate

When `maturity > 0`:
- `settle()` requires `env.ledger().timestamp() >= escrow.maturity`
- When `maturity == 0`: `settle()` succeeds immediately (no time gate)

`withdraw()` does **not** check maturity; it is a pull model for SME liquidity.

---

## Maturity extension guard (`update_ma

/* … truncated 4646 chars — edit only what you need near the top … */
