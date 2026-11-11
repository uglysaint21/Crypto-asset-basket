# Solana Portfolio Vault — Implementation Plan

**Status:** Security-remediated implementation plan; ready for phased implementation subject to explicit release gates  
**Date:** 2026-11-11  
**Repository:** `uglysaint21/Crypto-asset-basket`  
**Branch:** `design/solana-vault-architecture`  
**Design specification:** `docs/superpowers/specs/2026-11-11-solana-portfolio-vault-design.md`

## 1. Objective and delivery boundary

Implement the approved non-custodial Solana portfolio vault as an Anchor program, using permissionless staged operations, fixed portfolio weights, USDC deposits and redemptions, an SPL portfolio-share mint, oracle-free on-chain valuation, and tightly validated DEX CPI execution.

This plan covers the first audited-testnet-ready implementation milestone. It does not authorize mainnet deployment, real-fund use, a hosted keeper, a privileged investment manager, or a frontend as a prerequisite for protocol operation.

The two newly approved architectural decisions are binding design inputs:

1. **DEX execution:** Jupiter Metis on-chain Router as the primary route execution integration, behind a narrow adapter. Keep adapter boundaries modular for future direct Raydium CPMM and Orca Whirlpool integrations; do not add those adapters to v1 merely for breadth. Do not use Jupiter's managed Meta-Aggregator/RFQ execution path.
2. **Partial redemption recovery:** PDA-backed, non-transferable Recovery Tickets. After the initial seven-day recovery delay, the ticket owner may atomically convert an incomplete redemption into a ticket that owns the accumulated USDC and all unsold reserved assets. Anyone may continue valid swaps from ticket escrow. The owner may claim all USDC when liquidation completes, or explicitly elect to receive residual tokens in kind. Tickets never expire into protocol ownership and cannot be seized or redirected by an administrator.

## 2. Implementation principles

- **Tests first:** use red-green-refactor for every behavior and security invariant.
- **No guessed interfaces:** pin Anchor/Solana/toolchain versions and verify the exact Jupiter deployed program, CPI interface, IDL, account constraints, upgrade authority, downstream program behavior and token compatibility before implementing the adapter.
- **Fail closed:** unknown programs, mints, pools, token extensions, account layouts, routes or valuation states are rejected.
- **No arbitrary CPI:** route execution is constrained by a program-owned allowlist/configuration and strict account validation.
- **Actual balance deltas:** state transitions record actual input spent and output received, never caller-supplied quote claims.
- **Permissionless progress:** after an operation is opened, any party may submit a valid next step; no server key or keeper is required.
- **Explicit custody and accounting:** live vault balances, operation staging balances and ticket escrow balances are distinct and never double-counted.
- **No privileged asset control:** no admin withdrawal, discretionary mint, allocation mutation or user freeze path.
- **Incremental scope:** build and test accounting/state machinery before DEX integration.

## 3. Proposed repository structure

Confirm the existing repository layout before creating files. If it remains documentation-only, initialize a conventional Anchor workspace:

```text
programs/
  portfolio_vault/
    src/
      lib.rs
      state/
        mod.rs
        vault.rs
        asset.rs
        deposit.rs
        redemption.rs
        recovery_ticket.rs
      instructions/
        mod.rs
        initialize.rs
        deposit/
        redemption/
        recovery/
      accounting/
        mod.rs
        fixed_point.rs
        nav.rs
        shares.rs
        allocation.rs
      dex/
        mod.rs
        traits.rs
        jupiter_metis/
          mod.rs
          accounts.rs
          validation.rs
      errors.rs
      events.rs
  tests/
    accounting/
    state_machine/
    adversarial/
    dex/
  migrations/                 # only if required by selected Anchor version
  Anchor.toml
  Cargo.toml
  package.json                # test runner only if required
docs/
  superpowers/
    specs/
    plans/
```

The exact module split may be adjusted to the pinned Anchor version, but state, accounting, instruction validation and DEX adapter responsibilities must remain separated.

## 4. Phase 0 — Repository and toolchain baseline

1. Inspect the full repository, branch, existing files and history before modifying project code.
2. Select and pin mutually compatible versions of Rust, Solana/Agave CLI, Anchor, Rust crates and test tooling.
3. Add reproducible setup instructions and a minimal build/test command set.
4. Initialize the Anchor workspace only after recording the baseline.
5. Add CI for formatting, linting, unit tests and program build.

**Exit criteria**
- A clean checkout can reproduce the documented build.
- Toolchain versions are pinned.
- CI fails on formatting, lint, compile or unit-test failures.
- No protocol behavior has been implemented without tests.

## 5. Phase 1 — Pure accounting and configuration (TDD)

Implement pure functions before account instructions:

- Validate that target weights total exactly 10,000 basis points.
- Validate unique configured mints and supported token program IDs.
- Calculate per-asset target allocations with deterministic rounding.
- Implement checked fixed-point arithmetic and safe multiply/divide helpers.
- Define share decimal scale, virtual asset/share offsets, minimum initial deposit and minimum share issuance after arithmetic analysis.
- Calculate shares for subsequent deposits from pre-deposit NAV and share supply.
- Calculate proportional redemption quantities from a fixed share fraction.
- Define dust ownership, conservative rounding, recognized-balance ledger and unsolicited-transfer surplus policy.
- Model NAV liabilities and ensure pending operations/tickets are excluded or included exactly once according to state.

**Required tests**
- Zero NAV, zero supply, first deposit, very small deposits and dust.
- Overflow, underflow, division by zero and rounding boundaries.
- Allocation sum and per-slot rounding conservation.
- Multiple sequential deposits and redemptions.
- Proportional claims across all holdings.
- Invariant that pending staged assets are not counted as live NAV and as contribution simultaneously.

**Exit criteria**
- Pure accounting tests pass, including property-based/fuzz tests for arithmetic and conservation where practical.
- Formulas and rounding policy are documented and deterministic.

## 6. Phase 2 — Vault state, PDAs and share token

Implement initialization and core account constraints. Do not create a persistent global operation lock; use per-operation escrow, aggregate locked-share and per-asset reservation ledgers:

- `VaultState` with immutable policy/configuration hash, USDC mint, share mint, fixed allocation, exact asset mints/programs, operation sequence, aggregate locked shares, per-asset reserved balances and recognized-balance ledger.
- PDA authorities and token accounts for each configured asset.
- Standard SPL share mint controlled by a PDA; no freeze authority.
- Explicit token-account ownership/mint/program checks.
- No post-initialization arbitrary asset or allocation mutation instruction.

**Required tests**
- Correct PDA derivations and signer seeds.
- Wrong mint, token program, authority, owner and account substitution rejected.
- Duplicate assets and incorrect weight totals rejected.
- Share mint authority is program-controlled; freeze authority absent.
- No instruction can transfer vault assets to an arbitrary recipient.

**Exit criteria**
- Initialization and account-validation tests pass on local validator.
- State versioning and account-size strategy documented.

## 7. Phase 3 — NAV valuation boundary (oracle-free)

Implement a valuation interface independent of any one DEX adapter:

- Read only explicitly supported on-chain pool/state accounts.
- Validate account owner, discriminator/layout, mint pair, vaults, token programs and freshness/slot constraints.
- Apply per-asset liquidity and price-impact limits.
- Reject stale, inconsistent, manipulated or insufficient-depth valuation inputs.
- Use conservative valuation and deterministic integer math.
- Do not accept a caller-provided price or off-chain quote as an on-chain valuation input.

The NAV method must not use a single instantaneous spot quote. Each enabled asset must have on-chain time-weighted observations from at least two economically independent venues where available (initial observation-history baseline: 30 minutes), cross-venue and executable-sell-quote deviation checks, liquidity/price-impact bounds and a conservative haircut. Numeric per-asset limits and observation adapters are release gates based on measured liquidity and manipulation-cost analysis. If safe observations are unavailable, fail closed: do not finalize deposits or mint shares. No caller price, off-chain quote or single-pool fallback is allowed.

**Required tests**
- Wrong owner/layout/mints/pool vaults rejected.
- Stale or inconsistent pool state rejected.
- Low liquidity and excessive price impact rejected.
- Manipulated spot state / single-pool concentration scenarios.
- Valuation rounding and conservative haircut behavior.

**Exit criteria**
- Valuation rules are documented per configured asset.
- Adversarial valuation tests pass.
- If a safe valuation cannot be demonstrated for an asset, that asset is not enabled.

## 8. Phase 4 — Deposit state machine

Implement staged USDC deposits:

1. Open deposit and escrow USDC with caller min-output/expiry constraints.
2. Create an isolated operation PDA and escrow USDC. Do not take an authoritative NAV snapshot yet; it is taken at finalization.
3. Derive fixed allocation amounts; retain the configured USDC reserve portion.
4. Execute one asset swap per permissionless step into operation-specific staging accounts.
5. Record actual spend/output and mark each step complete exactly once.
6. Finalize only when all required steps are complete, snapshot remains valid and accounting checks pass.
7. In one atomic finalization, value the staged contribution and current live NAV using the same validated on-chain rules, move staged assets into live balances, update recognized balances and mint shares. Reject stale/unsafe valuation; no long-lived global lock is held.
8. At the immutable seven-day deadline from opening, anyone may convert the deposit into a Deposit Recovery Ticket holding all residual USDC and acquired tokens. No shares are minted. The depositor remains the sole beneficiary; permissionless swaps obey depositor-set execution floors, and the depositor may explicitly claim residual assets in kind.

**Deposit recovery is defined:** use a PDA-backed Deposit Recovery Ticket; do not assume an unwind route exists. The ticket holds all residual USDC and acquired tokens, preserves the depositor's claim, and never expires into protocol ownership.

**Required tests**
- Shares cannot mint before all steps complete.
- Failed CPI/min-output leaves balances and step state unchanged.
- Replayed step cannot spend twice.
- Staged assets never count as live NAV before finalization.
- Expired/stale snapshot cannot finalize.
- Partial expiry recovery conserves all deposited value.
- Competing operations cannot spend reserved balances.

**Exit criteria**
- End-to-end local-validator deposit tests pass for full, partial, failed and expired paths.
- Deposit preview values are explicitly informational; on-chain constraints are authoritative.

## 9. Phase 5 — Redemption and Recovery Ticket state machines

Implement standard USDC-only redemption:

1. Preflight candidate routes for every proportional asset quantity.
2. Open redemption, lock shares in PDA escrow, snapshot proportional quantities and reserve inventory.
3. Execute each asset-to-USDC swap independently into isolated redemption USDC escrow.
4. Track each completed step and actual balance deltas; failed steps remain retryable.
5. When all assets are liquidated, enforce aggregate minimum USDC, burn locked shares and transfer one USDC payment.
6. Permit cancellation and share unlock only before any liquidation has occurred.

Implement the approved Recovery Ticket transition:

1. Enforce the fixed seven-day delay from the defined recovery clock (clock source and exact start event must be documented).
2. Only the redemption owner may request conversion.
3. In one atomic instruction, move accumulated USDC and all unsold reserved assets into ticket-specific escrow, burn the locked shares, record the ticket claim and release the global NAV-operation lock.
4. Exclude ticket escrow from active vault NAV; it is a separate user liability/claim.
5. Allow anyone to execute validated asset-to-USDC swaps from ticket escrow.
6. Permit only the ticket owner to claim accumulated USDC after all residual assets are liquidated.
7. Store owner-approved per-asset minimum execution rates. Permissionless executors cannot lower them. Only the owner may update a floor, incrementing a terms version and emitting an event. The owner may explicitly elect in-kind claim of remaining token balances; this choice must not be the default.
8. Close the ticket only when all claim balances are zero. No expiry forfeiture, admin seizure or redirection.

**Required tests**
- Preflight fails without locking shares if any required route is unacceptable.
- Proportional reservation and share escrow correctness.
- Failed swap rollback; completed step replay protection.
- Normal completion burns shares and transfers exactly one USDC payment.
- Cancellation impossible after first successful sale.
- Recovery conversion rejected before the immutable seven-day expiry; any caller may trigger it after expiry, but only the recorded owner receives the ticket claim.
- Conversion atomically moves every residual asset/proceeds and burns shares.
- Operation lock releases only after successful ticket conversion.
- Ticket balances are never included in active NAV; pending redemption assets/proceeds remain included while locked shares remain in total supply and leave NAV atomically with share burn.
- Ticket swap proceeds cannot be double counted.
- Ticket owner alone can claim USDC, update execution floors, consent to a lower aggregate minimum, or choose in-kind.
- Ticket cannot be closed while any claim balance remains.
- No ticket can be confiscated, redirected or expired into protocol ownership.
- Property tests prove claim conservation through arbitrary sequences of successful/failed steps.

**Exit criteria**
- State-machine tests cover all terminal and retryable states.
- Invariants prove no double counting, no lost claim and no share/asset mismatch.

## 10. Phase 6 — Jupiter Metis Router adapter

This phase is gated on independent verification, not merely the existence of API documentation.

Verification checklist:
- Current deployed Router program ID on the target cluster.
- Exact CPI instruction interface and account ordering/privileges.
- Versioned IDL/source and reproducible account derivation where applicable.
- Program upgrade authority and upgrade process.
- Downstream DEX program IDs and route/account constraints.
- Supported SPL Token and Token-2022 behavior for each candidate asset.
- Remaining-account handling and CPI depth/compute limits.
- Whether the program can enforce all required destination, mint, authority, amount and minimum-output constraints.

Implementation:
- Define a narrow adapter interface used by deposit, redemption and ticket steps.
- Accept only configured Router program ID and supported downstream programs.
- Validate all source/destination token accounts, mint pairs, PDA signer, amount, minimum output and remaining accounts.
- Prevent arbitrary fee recipient, destination redirection, token substitution and unapproved CPI.
- Measure pre/post token balances and record actual deltas.
- Use caller-supplied route instructions only as untrusted candidates subject to full program validation.
- Keep route construction outside the trust boundary; no hosted route service is required for protocol correctness.

**Fallback rule:** if the exact Router CPI contract or account constraints cannot be verified and enforced, do not ship this adapter. Stop and present a direct, independently audited DEX adapter alternative for owner approval. Direct Raydium CPMM and Orca Whirlpool adapters remain future modular extensions, not assumed v1 functionality.

**Required tests**
- Valid route integration on local validator/fork with pinned program versions.
- Wrong router/downstream program, pool, mint, source, destination and signer rejected.
- Malicious remaining accounts and fee recipient substitution rejected.
- Slippage failure and insufficient output rollback.
- Actual balance delta accounting.
- Token-2022 transfer-fee/extension compatibility tests for each enabled asset.
- Compute/account-limit tests at realistic route complexity.

**Exit criteria**
- Pinned adapter documentation and exact verification evidence committed.
- Integration and adversarial tests pass against pinned program deployment.
- Independent review finds no arbitrary CPI or destination-redirection path.

## 11. Phase 7 — Security, operational and release validation

- Threat-model account substitution, signer spoofing, PDA seed mistakes, reentrancy-like CPI assumptions, arithmetic bugs, stale pool state, sandwiching, liquidity disappearance, token freeze/blacklist/transfer restrictions, transfer-fee extensions, rent griefing, operation-lock denial of service and malicious route accounts.
- Add property-based state-machine tests and fuzz tests for instruction/account parsing where practical.
- Test transaction size, compute budget, account count and CPI depth for the full configured basket.
- Test recovery when the primary route builder is unavailable; independently constructed valid routes must still be submit-able.
- Confirm that a stuck redemption can become a ticket after seven days and does not permanently block other vault operations.
- Document user-facing risks: token issuer/custody dependencies, no oracle, DEX liquidity risk, possible in-kind recovery, no guaranteed USDC conversion.
- Obtain an independent Solana program audit before mainnet.
- Revoke upgrade authority only after audit and deployment rehearsal, if immutability remains the chosen release posture.

**Release gates**
- All unit, integration, property and adversarial tests pass.
- No unresolved critical/high severity findings.
- All enabled assets have verified mints, token programs, issuer constraints, executable routes and minimum liquidity.
- All operational instructions are permissionless where specified.
- Mainnet deployment is a separate explicit decision and is not part of this plan.

## 12. Decisions still required before relevant code

The following are not silently assumed by this plan:

1. Exact mint addresses, issuer details and token program compatibility for every portfolio asset, including the unfinished global-equity, emerging-markets and individual-stock slots.
2. Per-asset liquidity thresholds, maximum price impact, route constraints and NAV valuation methodology.
3. Share decimals, initial share ratio and rounding/dust rules.
4. Per-asset numerical valuation/execution thresholds and virtual asset/share offsets, minimum initial deposit, share precision and dust constants.
5. Maximum concurrent operation/account limits and per-asset reservation bounds; no long-lived global lock is permitted.
6. Exact Jupiter Router deployment/CPI verification and whether it can satisfy the program's security boundary.
7. Exact seven-day recovery clock semantics (recommended: elapsed Solana Clock timestamp from the first successful redemption liquidation step, so the delay cannot be reset by retries).
8. V1 token policy: legacy SPL Token Program only by default; any exception for Token-2022 or active issuer authority requires separate review and explicit acceptance.

A decision that blocks safe accounting or asset custody must block that phase; it must not be filled with a guessed default.

## 13. Delivery order and approval gates

1. Owner approves this implementation plan.
2. Inspect and baseline the repository/toolchain.
3. Resolve foundational configuration and accounting decisions.
4. Implement pure accounting tests and functions.
5. Implement account constraints and vault initialization.
6. Implement valuation boundary.
7. Implement deposit state machine and its recovery path.
8. Implement redemption and Recovery Ticket state machines.
9. Verify and implement the Jupiter adapter.
10. Complete adversarial testing, independent review and release checklist.

No program code is changed until this plan is approved. After approval, implementation proceeds phase-by-phase with test-first commits and reviewable changes.


## 14. Security remediation requirements (supersede conflicting earlier plan wording)

The authoritative remediation decisions are documented in docs/superpowers/reviews/2026-11-11-solana-portfolio-vault-security-remediation.md and section 14 of the design specification.

### 14.1 Operation isolation and reservation accounting

- Remove the persistent global operation lock. Each deposit/redemption uses its own PDA and escrow accounts.
- Keep only short atomic state transitions; track aggregate locked shares and per-asset reserved balances.
- New redemptions use unlocked share supply and unreserved recognized inventory. Pending redemption escrow balances remain part of total NAV while corresponding shares remain in total supply.
- Queued requests do not snapshot NAV or reserve inventory until activation.
- Deposit staging is excluded from live NAV. Deposit share pricing uses current NAV and supply at finalization, atomically with staging transfer and mint.

### 14.2 Expiry and recovery

- All multi-step operations expire seven days after opening; retries cannot reset the deadline.
- Anyone may cancel an unstarted expired redemption; shares return only to its recorded owner.
- Anyone may convert a partially executed expired redemption into its owner's Recovery Ticket.
- Anyone may convert an incomplete expired deposit into its depositor's Deposit Recovery Ticket.
- Recovery tickets preserve every residual token and USDC unit, have no protocol/admin seizure or expiry, and are excluded from active NAV.
- Ticket executors cannot lower owner-set per-asset minimum execution rates. Only the owner may change floors. Below-minimum aggregate USDC claim requires explicit owner consent.

### 14.3 Valuation and share math

- No single-pool spot valuation. Use supported on-chain multi-venue TWAP observations (initial baseline: at least two independent venues and 30-minute observation history), current executable sell quote, deviation checks, liquidity/price-impact bounds and conservative haircut.
- If safe valuation is unavailable, deposit finalization/share minting fails closed. No off-chain quote or caller-supplied price fallback.
- Use virtual asset/share offsets, minimum initial deposit, minimum shares, checked wide-integer math, conservative rounding and a recognized-balance ledger.
- Unsolicited token transfers are unrecognized surplus and excluded from NAV; no admin sweep.

### 14.4 DEX CPI and token admission

- Jupiter /build is only an off-chain instruction builder. Verify and pin the exact on-chain program and CPI contract before implementation.
- Accept only a verified swap instruction; strictly validate program, discriminator, accounts, mints, token programs, PDA signer, amount, min output, downstream programs and actual balance deltas.
- Expose only operation-specific escrow accounts to CPI; never shared live vault token accounts.
- If safe validation cannot be proven, do not ship the Jupiter CPI integration.
- V1 portfolio assets default to legacy SPL Token Program with revoked mint/freeze authorities. Token-2022 and active issuer-authority exceptions require asset-specific review and explicit acceptance.

### 14.5 Upgrade and product semantics

- Development upgrade authority must be held by a secure multisig. Mainnet immutability after independent audit, reproducible build and deployment rehearsal is preferred.
- Allocation percentages are new-deposit weights, not continuously maintained target holdings. Display actual weight drift; no automatic rebalancing.

### 14.6 Revised release blockers

Do not enable an asset until exact mint, issuer/authority state, token program, supported extensions, executable route, independent on-chain observation venues, liquidity and per-asset thresholds are verified. Do not implement the Jupiter adapter until its exact deployment and CPI semantics are verified. Select virtual accounting constants through formal arithmetic tests. Obtain an independent audit before mainnet.
