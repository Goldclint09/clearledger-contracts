# Limitations

> **Updated:** 2026-10-20
> **Applies to:** ClearLedger protocol on Stellar Testnet only. There is no
> audited Mainnet deployment.

This document captures known limitations, testnet-specific constraints,
budget/cost estimates, unimplemented features, and edge cases that are not
yet handled. Behaviour described here is based on the current contract source,
not on an audit.

---

## Table of Contents

- [Testnet Limitations](#testnet-limitations)
- [Gas & Budget](#gas--budget)
- [Known Gaps](#known-gaps)
- [Unhandled Edge Cases](#unhandled-edge-cases)
- [Planned But Not Implemented](#planned-but-not-implemented)
- [Resolved](#resolved)
- [Related Issues](#related-issues)

---

## Testnet Limitations

### Friendbot-Derived Accounts

Testnet accounts are funded via Friendbot, which provides a fixed amount of
test XLM. All USDC on testnet is issued by the testnet USDC issuer and has no
real-world value. This means:

- **No real economic incentives** are at play — behaviour on testnet may not
  reflect mainnet dynamics.
- **Friendbot rate limits** can slow down testing. If `setup-testnet.sh`
  returns a `400` from Friendbot, the account is likely already funded
  (the error is benign).

### Ephemeral Validators

Stellar testnet validators may reset state periodically. Do not depend on
testnet contract state for production data.

### No Formal Audit

All contracts are unaudited. See [SECURITY.md](../SECURITY.md#audit-status)
for details.

### Deprecated Test CLI Commands

The `stellar contract` subcommand used in deploy scripts may change with CLI
upgrades. The repository pins `Rust 1.85.0` and the `wasm32v1-none` target,
but the Stellar CLI is expected to be at the latest version
(see [DEPLOYMENT.md](../DEPLOYMENT.md#prerequisites)).

---

## Gas & Budget

### Soroban Resource Model

Soroban charges fees based on a budget model: each operation consumes
CPU instructions, memory, and ledger I/O. The following are approximate
relative costs based on code analysis.

| Operation | Cross-Contract Calls | Token Transfers | CPU Instructions | Memory (Bytes) | Relative Budget |
|-----------|---------------------|-----------------|------------------|----------------|-----------------|
| `registry::register_issuer` | 0 | 0 | — | — | Very Low |
| `registry::revoke` | 0 | 0 | — | — | Very Low |
| `registry::get_profile_count` | 0 | 0 | — | — | Very Low |
| `registry::list_profiles` | 0 | 0 | — | — | Low (O(limit), ≤ 50) |
| `invoice::create` | 2 (`is_verified` ×2) | 0 | — | — | Low |
| `invoice::batch_create` | 2 per entry (`is_verified` ×2) | 0 | — | — | Low per entry, **up to 50× `create`** |
| `invoice::list_for_financing` | 0 | 0 | — | — | Low |
| `invoice::batch_list_for_financing` | 0 | 0 | — | — | Low per entry, **up to 50× `list_for_financing`** |
| `invoice::mark_funded` | 0 | 0 | — | — | Low |
| `invoice::repay` / `repay_partial` | 2 (`escrow::release_to_pool`, `pool::receive_repayment_with_refund`) | 2+ (buyer → escrow, escrow → pool, optional pool → buyer refund) | — | — | Medium |
| `invoice::repay_early` | 2 (`escrow::release_to_pool`, `pool::receive_repayment_with_refund`) | 2+ (buyer → escrow, escrow → pool, pool → buyer refund) | — | — | Medium |
| `invoice::trigger_default` | 1 (`handle_default`) | 0 | — | — | Medium |
| `pool::deposit` (before refactor) | 0 | 1 (LP → pool) | ~456,800 | ~73,400 | Low-Medium |
| `pool::deposit` (after `mint()`) | 0 | 1 (LP → pool) | 457,992 | 73,495 | Low-Medium |
| `pool::withdraw` (before refactor) | 0 | 1 (pool → LP) | ~450,500 | ~63,550 | Medium |
| `pool::withdraw` (after `burn()`) | 0 | 1 (pool → LP) | 451,634 | 63,660 | Medium |
| `pool::fund_invoice` | 8 (`get_funding_terms`, `get_issuer`, `get_buyer`, `get_funding_asset`, `is_verified` ×2, `lock`, `mark_funded`) | 1 (pool → escrow) | — | — | **High** |
| `pool::batch_fund_invoice` | 8 per entry | 1 per entry | — | — | **High** per entry, **up to 50× `fund_invoice`** |
| `pool::receive_repayment` | 0 | 0 | — | — | Low |
| `pool::handle_default` | 1 (`escrow::handle_default`) | 1 (escrow → pool) | — | — | Medium |
| `pool_factory::register_existing_pool` | 0 | 0 | 69,486 | 7,604 | Low |
| `escrow::lock` | 0 | 1 (pool → escrow) | — | — | Medium |
| `escrow::release_to_issuer` | 0 | 1 (escrow → issuer) | — | — | Medium |
| `escrow::release_to_pool` | 0 | 1 (escrow → pool, partial allowed) | — | — | Medium |
| `escrow::handle_default` | 0 | 1 (escrow → pool) | — | — | Medium |

Most mutating entry points above read the per-contract pause flag from
`clearledger_pause`, adding one instance-storage read per call. The flag lives
in each contract's own instance storage, so there is no single global pause
switch. Exceptions are covered in
[Pause Does Not Cover Every State-Changing Path](#pause-does-not-cover-every-state-changing-path).
That extra read also pushed a few long-running integration tests past the
default _test-host_ budget, which is why `pool_factory`'s test `setup()` lifts
that budget — see [Batch Operations](#batch-operations).

### Measured Benchmarks: `deposit()` and `withdraw()` (SEP-41 mint/burn refactor)

In preparation for SEP-41 token interface compliance, `deposit()` and `withdraw()`'s internal LP share storage updates were refactored from inline writes into shared internal `mint()` and `burn()` helpers. Resource measurements were taken via Soroban SDK test environment budget tracking (`env.budget()` in `test_gas_benchmark_deposit_and_withdraw` in `contracts/pool/src/test.rs`):

| Operation    | Implementation                                   | CPU Instructions | Memory (Bytes) | Delta (CPU)     | Delta (Memory) |
| ------------ | ------------------------------------------------ | ---------------- | -------------- | --------------- | -------------- |
| `deposit()`  | Inline `DataKey::LPShares` writes (pre-refactor) | 456,800          | 73,400         | Baseline        | Baseline       |
| `deposit()`  | Routed via `Self::mint()` (post-refactor)        | 457,992          | 73,495         | +1,192 (+0.26%) | +95 (+0.13%)   |
| `withdraw()` | Inline `DataKey::LPShares` writes (pre-refactor) | 450,500          | 63,550         | Baseline        | Baseline       |
| `withdraw()` | Routed via `Self::burn()` (post-refactor)        | 451,634          | 63,660         | +1,134 (+0.25%) | +110 (+0.17%)  |

The benchmark demonstrates negligible gas overhead (~0.25% CPU instruction delta from standard helper call frames) with zero regression to ledger write patterns or storage footprint.

### Budget Considerations

- `pool::fund_invoice` is the most expensive operation — it makes 8
  cross-contract calls (4 invoice reads, 2 registry verification reads,
  `escrow::lock`, and `invoice::mark_funded`) and performs 1 token transfer. On
  a congested testnet, this may exceed the per-ledger resource limit.
- Read-only view functions (`get_stats`, `get`, `get_profile`, etc.) incur
  minimal cost as they only read from storage.
- Index enumeration functions (`get_by_status`, `get_by_issuer`,
  `get_by_buyer`) are paginated (`page`, `page_size`, capped at
  `MAX_PAGE_SIZE`), so each call hydrates a bounded number of invoices no
  matter how many entries the index holds (issue #71). Callers page until an
  empty result is returned. The status index is append-only, so a page can
  return fewer than `page_size` invoices when it contains stale entries whose
  invoice has since moved to another status.
- `get_invoice_count_by_issuer` and `get_invoice_count_by_buyer` avoid that
  cost entirely: they read a single stored counter (`u32`) in O(1), so
  pagination and badge UIs should prefer them over `.len()` on the
  full-fetch views.
- `registry::list_profiles` is bounded per call: it returns at most 50
  addresses (`PageSizeExceeded` above that, mirroring the 50-entry batch cap)
  and costs O(1) in `limit`, not in the total number of registered profiles.
  Read `registry::get_profile_count(role)` once to size the final page, then
  walk `start = 0, limit, 2 * limit, …` until a page comes back short. Each
  enumerated address costs one persistent entry plus its TTL bump, so a page of
  50 is roughly 50 reads — indexers that want thousands of addresses should
  page off-chain rather than in one call.

### Batch Operations

`invoice::batch_create`, `invoice::batch_list_for_financing`, and
`pool::batch_fund_invoice` bound their input at `MAX_BATCH_SIZE` (50 entries)
with `BatchSizeExceeded` (`InvoiceError::28` / `PoolError::26`). The cap exists
so a caller cannot turn one transaction into an unbounded number of cross-contract
calls; a full 50-entry batch is the practical ceiling and can exceed the
per-ledger resource limit on a congested testnet. Callers that hit a budget
error should split the work across several transactions.

The three batch entry points deliberately differ in failure semantics, because
their failure modes differ:

| Entry point                | On a bad entry                      | Why                                                                                                                                                                  |
| -------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `batch_create`             | Reverts the whole batch             | Creation consumes the shared `Counter`; a partial success would burn counter values and persist a prefix the caller cannot identify without re-reading every invoice |
| `batch_list_for_financing` | Skips the entry, returns its ID     | The interesting failures are per-invoice state an issuer listing a backlog cannot avoid (already listed, missing attestation, revoked verification)                  |
| `batch_fund_invoice`       | Stops the batch, returns funded IDs | A liquidity/utilization stop is a normal outcome when an LP fills the pool; partial funding would drain the pool in a way no single `fund_invoice` could             |

`batch_fund_invoice` still reverts on _eligibility_ errors (not listed, already
funded by this pool, unverified issuer/buyer, asset mismatch), because those are
caller mistakes that should be fixed rather than silently skipped. Callers
reconcile the returned funded IDs against their input to see which entries a
capacity stop skipped.

### Storing the `History` Vector

`escrow::append_history` stores a growable `Vec<EscrowEvent>` under
`DataKey::History(invoice_id)`. Each event adds an element without
compaction. For heavily-used contracts, this vector may grow large enough
to exceed the per-entry size limit (~100 KB in practice).

---

## Known Gaps

### LP Withdrawal Yield Uses Aggregate Principal

The pool stores one cumulative principal amount per LP rather than deposit
lots. A partial withdrawal allocates that amount pro rata across the LP's
remaining shares. This is an average-basis approximation, not FIFO or HIFO
accounting; after deposits at different share prices it can attribute a
different yield amount to a particular withdrawal. Total USDC returned and
pool share ownership are unaffected, but the LP's cumulative `yield_earned`
report can differ based on the chosen lot-accounting policy.

For example, an LP deposits 1,000 USDC, the pool gains 2%, then the LP deposits
another 1,000 USDC. Withdrawing 500 shares returns about 510 USDC. FIFO would
attribute 500 USDC principal and 10 USDC yield, while aggregate-basis
accounting records about 504.95 USDC principal and 5.05 USDC yield. The
roughly 4.95 USDC difference is about 49.5% of the FIFO yield for this
withdrawal. The regression test in `contracts/pool/src/test.rs` covers this
scenario. Per-deposit lots would be required if the protocol needs a specific
FIFO/HIFO tax or reporting policy.

### Issuer Release Not Wired (Issue #56)

`pool.fund_invoice` locks the discounted principal in escrow through
`escrow.lock` but never calls `escrow.release_to_issuer`
(`contracts/pool/src/lib.rs`, `fund_one`). `release_to_issuer` requires
authorization from the configured pool, and the pool exposes no entry point
that calls it, so this is not recoverable by the admin or an off-chain bot:
**there is currently no reachable path to pay the issuer**.

On the normal repayment path, `invoice.repay`/`repay_partial` sends the buyer's
`face_value` to escrow and then calls `escrow.release_to_pool(face_value)`.
That transfers `face_value` to the pool and deletes the lock record, leaving the
originally locked principal in the escrow contract with no record to release
it. On the default path, `escrow.handle_default` returns the locked principal
to the pool. See [Issue #56](https://github.com/Stellar-Clear/clearledger-contracts/issues/56).

**Status:** Unresolved and higher priority than the other gaps in this section.
**What does not work:** Calling `escrow.release_to_issuer` manually from the
admin account fails because the escrow requires pool authorization. Fixing this
requires new pool/escrow logic (or an upgrade), not an operational workaround.

### Pool `set_max_utilization` Does Not Check the Stored Admin

`pool.set_max_utilization(env, admin, new_cap_bps)` validates
`new_cap_bps <= 10000` and calls `admin.require_auth()`, but never compares
`admin` with the stored admin (`contracts/pool/src/lib.rs`,
`set_max_utilization`). Any account can pass its own address, authorize the
call, and change the cap within `0..=10000`.

Consequences: an arbitrary caller can set the cap to `0` and block all new
funding, or raise it to `10000` and allow 100% utilization. The existing test
`test_set_max_utilization_requires_admin_authorization` clears mocked auths, so
it does not catch the missing stored-admin check. This is an authentication
bug, not intended permissionless behaviour.

### Pause Does Not Cover Every State-Changing Path

The pause flag is per contract and admin-controlled. It blocks the guarded
entry points, including withdrawals, repayments, LP share transfers, and
default settlement. It does **not** block:

- `invoice.cancel`, which is issuer-authenticated but has no
  `require_not_paused` call (`contracts/invoice/src/lib.rs`, `cancel`). An
  issuer can cancel a `Created` invoice while the invoice contract is paused.
- `invoice.upgrade` and `escrow.upgrade`, which are admin-only but not
  pause-guarded (`contracts/invoice/src/lib.rs`, `contracts/escrow/src/lib.rs`).
- Any `pool_factory` entry point; the factory has no pause mechanism.

There is no pause expiry or timelock, so an admin can freeze a contract
indefinitely. Because pause also blocks `withdraw`, pausing a pool is not a
safe way to let LPs exit; unpause first, then coordinate exits.

### Upgrades Are Immediate and Centralised

`invoice.upgrade` and `escrow.upgrade` replace the contract Wasm in place using
the stored admin (`update_current_contract_wasm`). There is no timelock, no
pause guard, and no on-chain approval. A compromised invoice/escrow admin can
replace the code and move escrowed funds or rewrite invoice state. The registry,
pool, and factory have no upgrade entry point; changing them requires deploying
new contracts and updating every integrator. See the upgrade note in
[DEPLOYMENT.md](../DEPLOYMENT.md#upgrades).

### Funding Asset Must Be a Well-Behaved SEP-41 Token

`pool` and `escrow` assume the funding asset moves exactly the requested amount
on `transfer`. Fee-on-transfer, rebasing, and hook-based tokens are not
supported: `pool.deposit` records the requested amount in `TotalDeposits` after
the transfer, so a short transfer overstates pool assets. A token issuer that
can freeze or blacklist accounts can also block deposits, withdrawals, funding,
and repayments.

The deploy scripts pass `USDC_ISSUER` directly as both `usdc_asset` and
`funding_asset`, so that variable must hold the USDC Stellar Asset Contract
(SAC) address (`C...`), not the `G...` issuer account shown in some examples.
The pool's share `decimals` is also not validated against the funding asset's
decimals.

### Direct Token Transfers Are Not Credited or Recoverable

Sending the funding asset directly to the pool or escrow does not update
`TotalDeposits`, create a lock record, or grant shares. There is no sweep or
admin recovery function for unsolicited transfers.

### No Accessor for `funded_amount` on Invoice Contract

`get_face_value`, `get_discount_bps`, `get_funding_terms`, and
`get_funding_asset` are exposed as per-field views, but the invoice contract has
no dedicated `get_funded_amount` accessor; callers read the full `Invoice`
through `get()`. The pool does expose `get_funded_amount(invoice_id)` for
invoices it has funded, which returns `None` before funding and after
repayment/default.

### No Duplicate Invoice Detection

The `create` function generates an invoice ID from a SHA-256 hash of
(issuer, buyer, face_value, due_date, counter, asset). The counter
guarantees uniqueness even for otherwise identical invoices, but there is
no user-visible warning when an issuer re-submits the same commercial
terms.

---

## Unhandled Edge Cases

### What Happens When…

| Scenario                                                               | Current Behaviour                                                                                   | Notes                                                                                                                                   |
| ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| LP deposits in a pool with 0 shares and 0 deposits                     | 1:1 share minting                                                                                   | Correct                                                                                                                                 |
| LP deposits when `total_shares > 0` but `total_deposits = 0`           | Division by zero would panic                                                                        | Cannot happen in practice (shares only minted alongside deposits)                                                                       |
| Buyer repays exactly on `due_date`                                     | Both `invoice.repay` and `trigger_default` are allowed                                               | `repay` has no time gate on `Confirmed`; `trigger_default` allows `now == due_date` (guard is `now < due_date`). Whichever lands first wins the race. |
| `confirm_delivery` called after invoice is already confirmed           | Panics with `InvalidStatusTransition` because status is `Confirmed` not `Active`                    | Correct — prevents replay                                                                                                               |
| `withdraw` when pool has funded invoices but no available liquidity    | Panics with `InsufficientLiquidity`                                                                 | Correct — prevents breaking the pool invariant                                                                                          |
| `set_max_utilization` is set to 0                                      | All `fund_invoice` calls fail (any utilization is above a 0 bps cap)                                | Any caller can do this, not just the admin, because the function does not check its `admin` argument against stored state                |
| Calling `revoke` on an unregistered address                            | Panics with `NotFound`                                                                              | Per spec                                                                                                                                |
| Calling `batch_register_issuers` with more than 50 entries             | Panics with `BatchSizeExceeded`                                                                     | Gas protection                                                                                                                          |
| A buyer repays _less_ than `face_value`                                | `invoice.repay_partial` accepts installments from `Funded`, `Active`, or `Confirmed`                | `invoice.repay` is the full remaining balance. Each partial payment is transferred buyer → escrow; settlement happens on the final payment. |
| An issuer creates an invoice with `due_date = current_time + 1` second | Invoice accepted                                                                                    | `create` requires `due_date > env.ledger().timestamp()` (strict); `now + 1` passes, `now` does not                                      |
| The pool has exactly 0 USDC balance after funding all deposits         | `withdraw` succeeds for unfunded amounts, but funded invoices are locked                            | Escrow holds funded USDC, pool holds only unfunded USDC                                                                                 |
| `trigger_default` while status is exactly at `due_date` (timestamp)    | Succeeds (subject to escrow's 60-second minimum lock window)                                        | The invoice guard is `now < due_date`; equality is allowed. If the invoice was funded less than 60s earlier, escrow reverts the call.     |

---

## Planned But Not Implemented

These features are tracked in the repository's issue tracker. Cross-links
are provided where available.

### UI / Frontend

- Event-driven Go indexer (see ClearLedger-app)
- Transaction simulation before signature

### Smart Contracts

- Multi-sig admin (3-of-5 Stellar signers)
- LP-governed invoice funding (stake LP tokens to vote on invoices). Design
  proposal: [NEW_DESIGN.md](NEW_DESIGN.md) (issue #718)
- Dynamic utilization-based interest rate model
- Swap-free multi-asset pools (USDC + XLM)
- On-chain governance for protocol parameters
- Timelocked / governed upgrades for the invoice and escrow contracts

### Tooling & Infrastructure

- Mainnet deployment support (currently testnet only)
- Formal verification or symbolic execution audit
- `cargo-soroban` integration for automated WASM budget reporting
- TypeScript SDK for contract interaction
- Automated fuzz testing across all core contracts

> **Note:** Buyer-driven partial repayments are supported by
> `invoice.repay_partial`. `invoice.repay` repays the remaining balance in one
> call. On the default side, `escrow.release_to_pool` accepts a
> `repayment_amount` and removes the lock record. See
> [Recovery and settlement paths](./THREAT_MODEL.md#recovery-and-settlement-paths).

---

## Resolved

These limitations were previously listed here but are no longer accurate for
the current source.

- **Status-index membership is O(1).** `DataKey::StatusMembership` exists in
  `contracts/invoice/src/types.rs`, and `extend_status_index` /
  `move_status_index` use it for duplicate detection and replayed-transition
  idempotency. Status rows are compacted with swap-and-pop in
  `remove_from_status_index`. The remaining performance cost is the scan in
  `remove_from_status_index`, which is O(n) in the size of the source status
  index; it is no longer a full scan on every membership check. This supersedes
  the earlier correction that claimed no such key existed.
- **Pool tracks per-LP yield accrual.** `pool::withdraw` writes
  `DataKey::LPYieldEarned(lp)`, and `pool::get_lp_position` returns the running
  yield figure. Yield is visible before an LP withdraws, subject to the
  aggregate-principal approximation above.
- **Partial buyer repayment is supported.** `invoice.repay_partial` accepts
  installments while the invoice is `Funded`, `Active`, or `Confirmed`;
  `invoice.repay` repays the remaining balance. The buyer path is no longer
  full-face-value only. Default-side `escrow.release_to_pool` accepts the
  `repayment_amount` it is given and removes the lock record.
- **Security findings #94, #96, #98, #103.** Addressed together in PR #124
  (`fix: address security issues #94, #96, #98, #103`).

---

## Related Issues

- #56 — Wire issuer release into `fund_invoice` (open; currently unreachable)
- #126 — Reconcile the threat model and limitations guide with contract
  behaviour (this revision)
- #285 — Event schema documentation
- #286 — Gas budget report
- #287 — Storage schema docs
- #294 — Testnet limitation docs
- #307 — General repo documentation
- #460 — LIMITATIONS.md stale date
- #831 / #835 — Old status-index documentation issues; current code has
  `DataKey::StatusMembership` and swap-and-pop compaction. The remaining O(n)
  removal scan is documented above.

Resolved items reference the PRs that closed them:

- Issue #67 — status index filtering (closed by PR #121)
- Issues #94, #96, #98, #103 — security (closed by PR #124)
- Partial default-side repayment (closed by PR #120)
