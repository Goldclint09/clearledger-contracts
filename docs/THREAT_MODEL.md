# Threat Model

> **Updated:** 2026-10-20
> **Applies to:** The four core Soroban contracts (`registry`, `invoice`,
> `escrow`, `pool`), the `pool_factory`, and the shared `clearledger-pause`
> library. **Stellar Testnet only.** There is no audited Mainnet deployment and
> nothing in this document should be read as Mainnet readiness.

This document describes the trust assumptions, authentication gates, pause and
recovery semantics, and attack surfaces of the ClearLedger protocol. It is
intended for security researchers, integrators, and contributors evaluating the
protocol's risk profile.

---

## Table of Contents

- [Trust Model](#trust-model)
- [Authentication Gates](#authentication-gates)
- [Pause & Recovery](#pause--recovery)
- [Token Assumptions](#token-assumptions)
- [Attack Vectors](#attack-vectors)
- [Centralization Risks](#centralization-risks)
- [Defence-in-Depth](#defence-in-depth)
- [Out-of-Scope](#out-of-scope)
- [Reporting](#reporting)

---

## Trust Model

### Roles

| Role | Description | Trust Level |
|------|-------------|-------------|
| **Registry admin** | Verifies/revokes profiles and can pause the registry. | **High** — controls all identity verification |
| **Invoice admin** | Re-wires contracts, sets the expiry window, pauses the invoice state machine, and can **upgrade the invoice Wasm**. | **High** — can replace invoice logic |
| **Escrow admin** | Pauses escrow and can **upgrade the escrow Wasm**. | **High** — can replace the code holding funds |
| **Pool admin** | Pauses the pool, sets the protocol fee/treasury, and can transfer pool ownership. | **High** — can freeze and reconfigure the pool |
| **Pool-factory admin** | Registers assets, deploys pools, and becomes the admin/treasury of every pool it deploys. | **High** |
| **Issuer** | Verified SME that creates and lists invoices. | Low — self-authenticates only |
| **Buyer** | Verified counterparty obligated to repay at maturity. | Low — self-authenticates only |
| **LP** | Deposits the funding asset and receives SEP-41 LP shares. | Low — self-authenticates only |
| **Anonymous caller** | Any Stellar account calling a permissionless function. | None — no auth required |

The bundled deploy scripts initialize every core contract with the same
deployer address as admin (`scripts/deploy.sh`, `scripts/deploy.ps1`). Unless a
multisig account is configured before deployment, one key controls all five
admin roles.

### Trust Boundaries

```
  ┌─────────────────────────────────────────────────────────────┐
  │                      Contract layer                         │
  │  ┌──────────┐  ┌─────────┐  ┌────────┐  ┌──────┐  ┌──────┐ │
  │  │ Registry │  │ Invoice │  │ Escrow │  │ Pool │  │Factory│ │
  │  └────┬─────┘  └────┬────┘  └───┬────┘  └──┬───┘  └──┬───┘ │
  │       └─────────────┴───────────┴───────────┴─────────┘     │
  │        Stored addresses are trusted after initialize()      │
  └─────────────────────────────────────────────────────────────┘
            ▲                    ▲                      ▲
            │                    │                      │
   Registry verification     SEP-41 token         LP / issuer / buyer
   (read by invoice/pool)    (funding asset)      (self-authenticated)
```

**Key trust assumptions:**

- All contracts trust the addresses stored at `initialize()` and the addresses
  later written by admin-gated setters. A wrong address can send funds
  irrevocably to the wrong contract or token.
- The funding asset is a well-behaved SEP-41 token. The token contract, and the
  issuer behind a classic Stellar asset, are trusted to move exactly the amount
  requested and not to freeze or blacklist the settlement contracts. See
  [Token Assumptions](#token-assumptions).
- `invoice` only accepts funding records from its configured pool
  (`mark_funded`), and `pool` only accepts settlement from its configured
  invoice contract (`receive_repayment`, `receive_repayment_with_refund`,
  `handle_default`).
- Agent attestations depend on a separately deployed agent registry
  (the `underwrite-contract` repository), configured through
  `invoice.set_agent_registry_contract`. That external contract's admin and
  agent set are part of the listing trust boundary.

---

## Authentication Gates

Every state-changing function enforces one of these patterns.

### Pattern A — Self-Auth (`address.require_auth()`)

The caller authenticates only themselves. No admin or third party is involved.

| Function | Contract | Who Auths |
|----------|----------|-----------|
| `register_issuer` | Registry | The issuer address |
| `register_buyer` | Registry | The buyer address |
| `update_profile`, `update_metadata` | Registry | The profile address |
| `create`, `batch_create` | Invoice | The issuer |
| `list_for_financing`, `batch_list_for_financing` | Invoice | Each invoice's issuer |
| `cancel` | Invoice | The issuer |
| `mark_shipped` | Invoice | The issuer |
| `confirm_delivery` | Invoice | The confirmer (issuer _or_ buyer) |
| `repay`, `repay_partial`, `repay_early` | Invoice | The buyer |
| `expire_listing` | Invoice | The caller, who must be the stored issuer _or_ admin |
| `deposit`, `withdraw` | Pool | The LP |
| `transfer`, `transfer_shares`, `approve`, `transfer_from` | Pool | Share holder / spender (SEP-41) |

### Pattern B — Admin-Auth (`admin.require_auth()`)

Only the stored admin address may call these functions, with the notable
`set_max_utilization` exception documented below.

| Function | Contract | Why |
|----------|----------|-----|
| `revoke`, `reinstate`, `verify_profile` | Registry | Controls verification status |
| `batch_register_issuers`, `batch_register_buyers` | Registry | Batch registration |
| `pause`, `unpause` | Registry, Invoice, Escrow, Pool | Emergency circuit breaker |
| `transfer_admin` | Registry | Single-admin key rotation (bypasses dual auth) |
| `set_pool_contract`, `set_escrow_contract`, `set_agent_registry_contract` | Invoice | Changes settlement wiring |
| `add_supported_asset`, `remove_supported_asset` | Invoice | Changes accepted funding assets |
| `set_expiry_window` | Invoice | Listing-expiry configuration |
| `upgrade` | Invoice, Escrow | Replaces contract Wasm |
| `set_protocol_fee` | Pool | Sets fee (max 2000 bps) and treasury |
| `register_asset`, `register_existing_pool` | Pool factory | Deploys/records pools |

`pool.set_max_utilization(env, admin, new_cap_bps)` is **not** in this table
because it does not compare its `admin` argument with the stored admin. It only
calls `admin.require_auth()`, so any caller can pass their own address and change
the cap (`contracts/pool/src/lib.rs`, `set_max_utilization`). This is a known
authentication gap, not an intended permissionless function; see
[LIMITATIONS.md](./LIMITATIONS.md#pool-set_max_utilization-does-not-check-the-stored-admin).

### Pattern C — Cross-Contract Auth (`contract.require_auth()`)

One contract authenticates as the caller of another contract's function.

| Function | Contract | Authenticated Caller | Why |
|----------|----------|---------------------|-----|
| `mark_funded` | Invoice | The configured pool contract | Prevents faking funding records |
| `mark_defaulted` | Invoice | The invoice's recorded funding pool | Prevents third-party default writes |
| `receive_repayment` | Pool | The configured invoice contract | Prevents faking repayment accounting |
| `receive_repayment_with_refund` | Pool | The configured invoice contract | Same as above |
| `handle_default` | Pool | The configured invoice contract | Prevents premature default recovery |
| `lock` | Escrow | The configured pool contract | Only the pool can lock funds |
| `release_to_issuer` | Escrow | The configured pool contract | Only the pool can release to the issuer |
| `release_to_pool` | Escrow | The configured pool contract | Only the pool can reclaim escrowed funds |
| `handle_default` | Escrow | The stored admin **or** the configured pool contract | Returns locked funds to the pool |

`invoice.mark_funded` additionally checks that the supplied `pool_address`
equals `DataKey::PoolContract` before calling `require_auth()`. `escrow` checks
that the stored issuer matches the `issuer` argument for
`release_to_issuer`.

### Pattern D — Dual-Auth (two `require_auth()` calls)

| Function | Contract | Signers |
|----------|----------|---------|
| `transfer_ownership` | Registry, Invoice, Pool | Current admin + new admin |

This is **not** universal:

- `initialize` requires only the incoming admin's authorization, not a second
  party. It is a single-auth bootstrap, protected by the already-initialized
  check.
- `escrow` has no ownership-transfer entry point. Its admin is fixed at
  `initialize` unless the contract is upgraded.
- `pool_factory` has no ownership-transfer entry point.
- `registry.transfer_admin(new_admin)` requires only the current admin and
  deliberately bypasses the dual-auth guarantee of `transfer_ownership`
  (`contracts/registry/src/lib.rs`, `transfer_admin`). A compromised registry
  admin can therefore hand the registry to an arbitrary address without that
  address signing.

### Pattern E — Permissionless (no caller auth)

**State-changing functions.** These are the calls with real trust
implications. Any Stellar account can invoke them.

| Function | Contract | Trust Implication |
|----------|----------|-------------------|
| `fund_invoice` | Pool | Commits pool capital to any `Listed` invoice that passes on-chain eligibility checks (asset match, registry verification, liquidity, utilization cap). LPs cannot veto individual invoices. |
| `batch_fund_invoice` | Pool | Same as `fund_invoice`, up to 50 entries, stopping when pool capacity is exhausted. |
| `trigger_default` | Invoice | At or after `due_date`, any caller can force a default. This calls `pool.handle_default` and `escrow.handle_default`, returns locked principal to the pool, and moves the invoice to `Defaulted`. A defaulted invoice can no longer be repaid through `repay`/`repay_partial`/`repay_early`. |
| `submit_attestation` | Invoice | Anyone may relay a signed attestation. Trust comes from the secp256k1 signature recovering to an active agent in the external agent registry, not from the caller. One attestation per invoice. |

`clearledger-pause` and `clearledger-ttl` are shared libraries, not deployed
contracts. `pool_factory` exposes no pause switch.

**Read-only views.** All views are permissionless and remain callable while a
contract is paused:

- **Registry:** `get_profile`, `is_verified`, `get_verification_status`,
  `get_profile_count`, `list_profiles`, `get_admin`.
- **Invoice:** `get`, `get_status`, `has_status_membership`, `get_face_value`,
  `get_remaining_balance`, `get_repaid_amount`, `get_discount_bps`,
  `get_funding_terms`, `get_funding_asset`, `get_issuer`, `get_buyer`,
  `get_due_date`, `get_by_status`, `get_by_issuer`, `get_by_buyer`,
  `get_invoice_count_by_issuer`, `get_invoice_count_by_buyer`, `get_counts`,
  `get_expiry_window`, `get_attestation`, `is_supported_asset`,
  `get_supported_asset_count`, `get_admin`, `get_registry_contract`,
  `get_pool_contract`, `get_escrow_contract`, `get_agent_registry_contract`.
- **Escrow:** `get_locked`, `get_locked_at`, `get_history`, `get_usdc_asset`,
  `get_admin`, `get_pool_contract`.
- **Pool:** `get_stats`, `get_lp_position`, `get_utilization_rate`,
  `get_funding_asset`, `get_usdc_asset`, `get_admin`, `get_fee_bps`,
  `get_treasury_address`, `get_protocol_fee_bps`, `get_treasury`,
  `get_invoice_contract`, `get_escrow_contract`, `get_suggested_discount_bps`,
  `get_funded_amount`, `preview_deposit`, `balance`, `total_supply`,
  `allowance`, `name`, `symbol`, `decimals`.
- **Pool factory:** `get_pool_for_asset`, `get_aggregate_stats`,
  `list_assets`.

---

## Pause & Recovery

### Pause semantics

Each core contract has its **own** pause flag, stored in that contract's
instance storage under `clearledger_pause::PauseState::Paused`. The library is
shared; the state is not global. Pausing the pool does not pause escrow,
invoice, or registry.

- `pause()` and `unpause()` are admin-only in `registry`, `invoice`, `escrow`,
  and `pool`. An absent flag means unpaused; a freshly deployed contract starts
  live.
- `unpause()` is deliberately not guarded by `require_not_paused`, so a paused
  contract can always be resumed by its admin.
- While paused, read-only views still work. State-changing entry points revert
  with `ContractPaused`.
- Pause is **all-or-nothing**: it also blocks withdrawals, repayments, funding,
  LP share transfers, and default settlement. It cannot be used to stop only
  new deposits or only new funding.
- Pause has no expiry or timelock. A malicious or compromised admin can pause
  indefinitely, and a paused escrow/pool can hold user funds until unpause.

Two state-changing paths are **not** covered by the pause flag:

- `invoice.cancel` is issuer-authenticated but does not call
  `require_not_paused` (`contracts/invoice/src/lib.rs`, `cancel`). An issuer can
  still cancel a `Created` invoice while the invoice contract is paused.
- `invoice.upgrade` and `escrow.upgrade` are admin-only and do not call
  `require_not_paused`. An admin can replace contract code while paused.

`pool_factory` has no pause mechanism at all, so pool deployment and
registration continue while the core contracts are paused.

### Recovery and settlement paths

| Situation | On-chain path | What actually happens |
|-----------|---------------|-----------------------|
| Buyer repays in full | `invoice.repay` → `invoice.repay_partial` → `escrow.release_to_pool` → `pool.receive_repayment_with_refund` | Buyer transfers `remaining_balance` to escrow; escrow sends `face_value` to the pool; pool credits the spread as yield and refunds any pro-rated early-payment amount to the buyer. |
| Buyer repays partially | `invoice.repay_partial` | Buyer transfers `amount` to escrow; invoice stays `Funded`/`Active`/`Confirmed` and tracks `repaid_amount`/`remaining_balance`. Full repayment settles as above. |
| Buyer repays before maturity | `invoice.repay_early` | Only for `Confirmed` invoices before `due_date`; the pool earns only the time-pro-rated discount and the rest is refunded to the buyer. |
| Invoice reaches maturity | `invoice.trigger_default` (permissionless) | At `now >= due_date`, calls `pool.handle_default`; escrow returns the locked principal to the pool; invoice becomes `Defaulted`; `TotalFunded` falls, `TotalDeposits` is unchanged, and `TotalLossRealised` is incremented. |
| Listing expires | `invoice.expire_listing` (issuer or admin) | At `now >= listed_at + expiry_window`, moves `Listed` → `Expired`. |
| Invoice no longer needed | `invoice.cancel` (issuer) | Moves `Created` → `Cancelled`; not pause-guarded. |
| Admin needs to fix code | `invoice.upgrade`, `escrow.upgrade` (admin) | Replaces the Wasm in place; storage and address stay the same. No timelock, no pause guard, and no on-chain review. |

There is **no admin rescue function** for user funds. The only direct admin
escape hatch is `escrow.handle_default(invoice_id, admin)`, which after the
60-second lock grace period returns the locked principal to the configured pool
and deletes the lock record. It does not update the invoice or pool accounting,
so using it outside the normal `invoice.trigger_default → pool.handle_default`
flow can desynchronise state.

### Default timing details

- `invoice.trigger_default` accepts `now >= due_date`; the equality boundary is
  allowed (`contracts/invoice/src/lib.rs`, `trigger_default`).
- `escrow.handle_default` separately requires
  `now - record.locked_at >= DEFAULT_MIN_LOCK_SECONDS` (60 seconds). If an
  invoice's `due_date` arrives less than 60 seconds after funding, default
  reverts until that escrow grace period elapses, even though the invoice
  due-date gate has passed.
- Once an invoice is `Defaulted`, the buyer-facing repayment functions reject
  it. There is no post-default cure window.

### Issuer release gap

`pool.fund_invoice` locks the discounted principal in escrow but never calls
`escrow.release_to_issuer`. `release_to_issuer` requires pool authorization and
the pool exposes no entry point that calls it, so the issuer payout path is not
merely "manual" — it is currently unreachable. On a normal repayment, escrow
releases `face_value` (funded principal plus buyer repayment) to the pool and
deletes the lock record; the locked principal remains in escrow with no record
to release it. On default, `escrow.handle_default` returns that same principal
to the pool. See
[LIMITATIONS.md](./LIMITATIONS.md#issuer-release-not-wired-issue-56).

---

## Token Assumptions

- `pool` and `escrow` interact with the funding asset through
  `soroban_sdk::token::Client` (`transfer`, and in `invoice.repay_partial`,
  `try_transfer`). The asset must implement SEP-41 semantics and move exactly
  the requested amount.
- **Fee-on-transfer, rebasing, and hook-based tokens are not supported.**
  `pool.deposit` credits `TotalDeposits` with the requested `usdc_amount` after
  calling `transfer`; if the token delivers less than requested, pool
  accounting overstates assets.
- A token issuer that can freeze or blacklist accounts can block deposits,
  withdrawals, funding, and repayments. The contracts have no fallback asset or
  admin override for a frozen token.
- The funding asset address is fixed per pool instance at `initialize` and is
  not upgradeable. `invoice` has an admin-managed allow-list of funding assets
  (`add_supported_asset`/`remove_supported_asset`); removing an asset prevents
  new invoices but does not affect existing ones.
- `pool` issues SEP-41 LP shares itself. Share `decimals` is supplied at
  `initialize` and is not validated against the funding asset's decimals. The
  deploy scripts use 7 decimals for the USDC pool.
- The deploy scripts pass the `USDC_ISSUER` environment variable directly as
  `usdc_asset`/`funding_asset`, despite the name. It must be the Stellar Asset
  Contract (SAC) address for USDC, not the `G...` issuer account. See
  [DEPLOYMENT.md](../DEPLOYMENT.md#environment-variables).

---

## Attack Vectors

### 1. Admin Key Compromise

**Risk:** Admin keys control pause, configuration, ownership transfer, and — for
invoice and escrow — arbitrary code upgrades. The bundled deploy script gives
all roles to one deployer key, so a single compromise can affect the entire
system.

**What is actually protected:**

- `registry`, `invoice`, and `pool` `transfer_ownership` require both the
  current and new admin to sign.
- `registry.transfer_admin` is a deliberate single-admin escape hatch, so the
  dual-auth protection can be bypassed by the current admin.
- `escrow` and `pool_factory` have no ownership-transfer entry point.
- An admin cannot directly call `escrow.release_to_issuer` or
  `release_to_pool`; those require the configured pool contract.
- Direct token transfers out of the pool or escrow are not available as an
  admin function.

**What is not protected:**

- `invoice.upgrade` and `escrow.upgrade` let the admin replace the contract
  code with any installed Wasm. That code can move escrowed funds or rewrite
  invoice state. There is no timelock, pause guard, or on-chain approval.
- The registry admin can revoke any profile, and `transfer_admin` can hand the
  registry to an arbitrary address.
- The pool admin can pause the pool indefinitely and set the protocol fee and
  treasury (fee capped at 2000 bps).
- `set_max_utilization` can be called by any account because the function does
  not validate its `admin` argument against stored state.

**Mitigation:** Configure admin as a multisig account before deploying, keep
signer keys separate, and treat invoice/escrow upgrades as the highest-risk
operation. Multisig is an account-level control, not an on-chain timelock.

### 2. Cross-Contract Spoofing

**Risk:** A malicious contract could try to call another contract's
cross-contract entry point and fake funding, repayment, or default state.

**Mitigation:**

- Contract addresses are written during `initialize()` and can only be changed
  by the relevant admin (`invoice.set_pool_contract`, `set_escrow_contract`,
  `set_agent_registry_contract`; `pool`/`escrow` have no re-wiring setters).
- `invoice.mark_funded` rejects any `pool_address` that is not the configured
  pool before requiring auth.
- `pool.receive_repayment`, `receive_repayment_with_refund`, and
  `handle_default` require auth from the configured invoice contract.
- `escrow.lock`, `release_to_issuer`, and `release_to_pool` require auth from
  the configured pool contract.
- The invoice's recorded `funding_pool` is used for `mark_defaulted`, so only
  the pool that funded an invoice can persist its default status.

### 3. Dust-Attack on Pool Shares

**Risk:** After the pool accrues yield, the share price rises above 1.0. A tiny
deposit can round down to 0 shares while the funding asset is still transferred
into the pool, donating the depositor's funds to existing LPs.

**Mitigation:** `deposit()` rejects any deposit that would mint 0 shares with
`PoolError::MinimumDeposit` (`#14`), and `withdraw()` rejects a redemption that
would return 0 units. The deposit check runs before the token transfer.

### 4. Replay of State Transitions

**Risk:** An attacker could try to call state-transition functions multiple
times on the same invoice (for example, double-funding or double-defaulting).

**Mitigation:** Every state transition checks the current status. Relevant
guards:

- `list_for_financing`: `Created` only.
- `mark_funded`: `Listed` only; the pool also keeps a `FundedInvoice` entry and
  checks it before any cross-contract read.
- `mark_shipped`: `Funded` only.
- `confirm_delivery`: `Active` only; each party can confirm once, and the
  invoice reaches `Confirmed` only when both have confirmed.
- `repay` / `repay_partial`: `Funded`, `Active`, or `Confirmed` only.
- `repay_early`: `Confirmed` only, strictly before `due_date`.
- `trigger_default`: `Funded`, `Active`, or `Confirmed` only, at or after
  `due_date`.
- `expire_listing`: `Listed` only.
- `cancel`: `Created` only.
- `escrow.lock`: rejects an existing lock with `EscrowError::AlreadyLocked`.
- `escrow.release_to_issuer`, `release_to_pool`, and `handle_default`: the lock
  record is removed on success, so a second call fails with `NotFound` (or
  returns `false` from `handle_default` when no record exists).

### 5. Integer Over/Underflow

**Risk:** Arithmetic errors in amount calculations could lead to incorrect fund
allocations.

**Mitigation:**

- Amounts are `u128`; the release profile enables `overflow-checks = true`
  (`Cargo.toml`).
- `receive_repayment` validates `amount >= funded_amount`.
- `receive_repayment_with_refund` validates `refund <= amount - funded_amount`.
- `withdraw` validates `shares <= lp_shares` and
  `usdc_to_return <= available_liquidity`.
- `pool.handle_default` uses `checked_sub` for `TotalFunded` and
  `ActiveInvoiceCount`; `TotalDeposits` is intentionally left unchanged so
  returning escrowed principal does not double-count the loss (issue #439).
- Discount and early-repayment math uses `checked_mul`/`checked_add` and
  `saturating_sub` where a floor of zero is intended.

### 6. Front-Running on Invoice Expiry

**Risk:** An attacker could front-run an `expire_listing` call to fund an
invoice that would otherwise expire.

**Mitigation:** `expire_listing` succeeds only when
`current_time >= listed_at + expiry_window` and only for the stored issuer or
admin. A funder can still call `fund_invoice` in the same ledger before the
expiry transaction. This race is accepted because funding is permissionless;
expiry does not reserve the invoice for anyone.

### 7. Utilization Cap Bypass

**Risk:** The utilization cap bounds how much of the pool can be deployed.
`set_max_utilization` rejects values above 10000 bps, but it does not check that
its `admin` argument is the stored admin. Any account can therefore lower the
cap (including to 0, blocking all funding) or raise it to 10000.

**Mitigation:** The `<= 10000` bound prevents over-leveraging above 100%, but it
does **not** prevent an arbitrary caller from changing the cap within that
range. This is a known authentication gap; treat the cap as untrusted until the
function is fixed.

### 8. Batch Registration Overflow

**Risk:** An attacker could submit a very large batch of issuers to exhaust gas
or storage.

**Mitigation:** `batch_register_issuers` and `batch_register_buyers` cap entries
at 50 and panic with `RegistryError::BatchSizeExceeded` otherwise. Invoice and
pool batch entry points use the same 50-entry cap.

### 9. Permissionless Funding and Default

**Risk:** Funding and default are not admin-gated. Any caller can commit pool
capital to an eligible listed invoice, and any caller can default an invoice at
or after its due date.

**Implications:**

- LPs have no on-chain per-invoice consent; they rely on the issuer/buyer
  verification checks, the attestation gate, the discount cap, and the
  utilization cap.
- A default is irreversible for the buyer: once `trigger_default` succeeds,
  `repay`/`repay_partial`/`repay_early` reject the invoice. There is no grace
  period after `due_date` beyond the escrow's 60-second minimum lock window.
- This is the intended settlement model for the Testnet deployment, not a
  governance upgrade path. Any change to funding approval or default timing
  requires new contract logic.

---

## Centralization Risks

| Risk | Current State | Notes |
|------|---------------|-------|
| Single admin key | Each contract has one admin; deploy scripts use one deployer for all | Configure a multisig account before any non-test deployment |
| No on-chain timelock | Invoice/escrow upgrades and pause changes execute immediately | Only account-level multisig/threshold delays |
| Admin-gated funding | **No** — `fund_invoice` is permissionless | LPs cannot veto individual invoices |
| Emergency pause | Implemented per contract, admin-only, indefinite | Blocks withdrawals and repayments too |
| Upgradeable code | Invoice and escrow only | Registry, pool, and factory require redeployment |
| Registry ownership escape hatch | `registry.transfer_admin` is single-auth | Bypasses `transfer_ownership` dual auth |
| Utilization cap auth | `set_max_utilization` accepts a caller-supplied admin | Any caller can change the cap |
| Centralised registry | Admin controls all verification/revocation | No on-chain governance |
| Factory control | Factory admin deploys pools and is their initial admin/treasury | Factory has no pause/upgrade/ownership transfer |

### Blast Radius: Admin Key

If an admin key is compromised or malicious:

| What They Can Do | What They Cannot Do |
|-----------------|---------------------|
| Upgrade invoice and escrow to arbitrary installed Wasm | Upgrade registry, pool, or factory (no upgrade entry point) |
| Pause any of the four core contracts indefinitely | Unpause someone else's contract (admin-only, but they are the admin) |
| Revoke or reinstate any registry profile | Directly call `escrow.release_to_issuer`/`release_to_pool` (pool auth required) |
| Hand the registry to an arbitrary address via `transfer_admin` | Transfer escrow or factory ownership (no such entry point) |
| Change the pool protocol fee/treasury (fee ≤ 2000 bps) | Set the fee above 2000 bps |
| Change the pool utilization cap | Set the cap above 10000 bps |
| Call `escrow.handle_default` after 60s to return locked principal to the pool | Send escrowed funds directly to themselves through a normal escrow function |
| Deploy new pools through the factory and become their admin | Change an existing pool's asset or wiring (no pool re-wiring setters) |

The upgrade capability dominates this table: an escrow admin can replace the
escrow code and then move funds through the new code, so a compromised escrow
admin key should be treated as a total-loss scenario for escrowed balances.

---

## Defence-in-Depth

### Separation of Concerns

- **Registry** holds verification state and no funds.
- **Escrow** holds the funding asset but releases it only for the configured
  pool (or returns it to the pool on default).
- **Pool** manages LP accounting and share issuance, and holds unfunded
  liquidity.
- **Invoice** is a lifecycle state machine with no fund custody; it coordinates
  token transfers between buyer, escrow, and pool.
- **Pool factory** deploys pools and records asset-to-pool mappings.

### TTL Management

All storage TTL bumps go through `clearledger-ttl` constants:
`THRESHOLD = 500_000` ledgers and `EXTEND_TO = 2_000_000` ledgers
(`contracts/ttl/src/lib.rs`). Persistent entries are extended on write; instance
storage is extended by each contract's `extend_instance_ttl()` helper on
state-changing calls.

See [STORAGE.md](./STORAGE.md) for the storage key schema. That document's
TTL table is older than the shared constants; the values above are the ones the
contracts actually use.

### Dependencies

The contracts depend on `soroban-sdk`, the shared `clearledger-pause` and
`clearledger-ttl` crates, the configured SEP-41 funding token, the registry, and
— only when attestation-gated listing is used — the external agent registry.
There are no price oracles, bridges, or off-chain relayers required for
settlement.

---

## Out-of-Scope

The following are explicitly outside this threat model:

- **Stellar protocol-level attacks** (consensus, validator compromise, etc.) —
  report to [Stellar Development Foundation](https://stellar.org).
- **Freighter / wallet vulnerabilities** — report to the respective wallet team.
- **Frontend phishing or social engineering** — users are responsible for
  verifying transaction details before signing.
- **Off-chain indexer downtime** — the contracts function independently of the
  indexer; missed events do not affect fund safety.
- **The external agent registry** (`underwrite-contract`) — its own admin,
  agent set, and verification logic are outside this repository's boundary.
- **Testnet-only risks** (Friendbot dust, ephemeral validators) — see
  [LIMITATIONS.md](./LIMITATIONS.md).

---

## Reporting

Report vulnerabilities through this repository's private security reporting
channel. See [SECURITY.md](../SECURITY.md) for the disclosure policy and
response timeline.
