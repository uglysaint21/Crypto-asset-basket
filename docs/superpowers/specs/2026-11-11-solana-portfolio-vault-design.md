# Solana Portfolio Vault — Design Specification

**Status:** Architecture approved by project owner; implementation plan pending approval  
**Date:** 2026-11-11  
**Repository:** `uglysaint21/Crypto-asset-basket`

## 1. Intent

Build a non-custodial, on-chain Solana portfolio vault. Users deposit USDC at any time. The program invests deposits according to fixed target weights, holds the resulting tokenized assets in PDA-controlled token accounts, and issues a transferable SPL portfolio share token. A user may redeem shares for one USDC payment; the vault liquidates the corresponding proportional quantities of its holdings through supported Solana DEX programs.

The protocol must not depend on external price oracles, a hosted investment server, an administrator who can move assets, or an administrator who can change investment policy.

## 2. Decisions supplied by the project owner

### Target portfolio weights

| Slot | Intended exposure | Weight | Verification state |
|---|---|---:|---|
| 0 | SPYx — S&P 500 | 10% | Exact mint and route must be re-verified |
| 1 | QQQx — Nasdaq-100 | 10% | Exact mint and route must be re-verified |
| 2 | Global equity ETF | 15% | Exact Solana token not yet selected |
| 3 | Emerging-markets ETF | 10% | Exact Solana token not yet selected |
| 4 | PAXG — gold | 20% | Exact mint and route must be re-verified |
| 5 | USDC reserve | 15% | USDC mint must be network-configured |
| 6 | Individual tokenized stocks basket | 10% | Constituent mints not yet selected |
| 7 | COPX | 5% | Exact mint and executable liquidity must be re-verified |
| 8 | USOon | 5% | Exact mint and executable liquidity must be re-verified |
| | **Total** | **100%** | |

Tickers are labels, never asset identifiers. Production configuration must use exact mint public keys, token program IDs, issuer information, and supported DEX route definitions. Unverified categories must not be silently replaced with another token. The program must not permit arbitrary user-supplied mints into the portfolio.

### Protocol requirements

- Deposits may be initiated at any time.
- Deposit asset is USDC.
- Deposit proceeds are allocated to fixed weights.
- Share token is a standard transferable SPL token.
- Withdrawals settle as one USDC payment.
- Withdrawal liquidation is proportional across portfolio holdings, including the USDC reserve.
- No external oracle programs or price-feed accounts.
- DEX pool state and actual swap execution may be used for valuation and execution protection.
- No automatic price-driven rebalancing.
- No privileged administrator may withdraw assets, arbitrarily mint shares, freeze users, or change allocations.
- The program should be immutable after audit and deployment; upgrade authority revocation is a deployment decision.

## 3. Goals, non-goals and anti-goals

### Goals

- Correct share accounting for deposits and redemptions.
- PDA-controlled custody of portfolio assets.
- Deterministic, fixed-weight investment policy.
- Protected on-chain DEX valuation without external oracles.
- Permissionless continuation of long-running operations where transaction limits require multiple transactions.
- Clear accounting for pending deposits, pending redemptions, reserved assets, fees and failed operations.

### Non-goals for the first implementation

- Frontend, hosted API, portfolio analytics service or off-chain investment manager.
- Dynamic asset selection or discretionary rebalancing.
- Permissionless arbitrary token listing.
- Tokenized asset issuer verification by the smart contract; issuer/backing due diligence remains an off-chain deployment prerequisite.
- Guaranteed execution when DEX liquidity is unavailable.
- Guaranteed USDC withdrawal settlement in one Solana transaction.

### Anti-goals

- No external oracle dependency.
- No admin key with custody or investment-policy powers.
- No arbitrary minting or burning of portfolio shares.
- No silent substitution of assets when a swap fails.
- No valuation based only on a caller-provided price or a single unprotected spot quote.

## 4. Approaches considered

### A. Fully atomic deposit and withdrawal

All swaps, accounting changes, share mint/burn and transfers occur in one transaction.

**Advantages:** simple state machine; transaction either fully succeeds or fully reverts.  
**Disadvantages:** a portfolio with many assets and DEX CPIs can exceed transaction account, compute or size limits. It is brittle as the asset basket grows.

### B. Permissionless staged execution

A request creates an on-chain operation state. Anyone can execute the next deterministic swap step. Shares are minted only when a deposit completes; redemption shares are locked and burned only when liquidation completes and USDC is paid.

**Advantages:** supports larger baskets; no privileged operator; retryable when liquidity or compute constraints temporarily prevent a step.  
**Disadvantages:** more complex accounting, operation locking, timeout/refund behavior and reserved-balance management.

### C. Hybrid atomic-first with staged fallback

Attempt all steps atomically, otherwise use a staged process.

**Advantages:** convenient when the full basket fits in one transaction.  
**Disadvantages:** two execution paths substantially increase testing and audit scope; fallback behavior can be difficult to reason about.

**Recommendation:** Approach B, permissionless staged execution, for the first production-oriented architecture. It is the most robust fit for the number of assets and direct DEX integrations. A small atomic path may be considered later only if it reuses the same accounting invariants.

## 5. Architecture

One Anchor program owns the vault state and PDA authorities. It maintains:
- A USDC vault token account.
- One vault token account per configured asset mint.
- An SPL share mint whose mint authority is a program PDA.
- Operation accounts for pending deposits and redemptions.
- Reservation/accounting fields that prevent concurrent operations from spending the same inventory.

DEX integrations are direct Solana CPIs through audited adapters. The first adapter(s) must be explicitly selected and pinned to known program IDs and account layouts. A hosted aggregator is not a mandatory dependency. A frontend or indexer may submit transactions and display events but is never authoritative for balances or permissions.

## 6. State and instruction boundaries

Proposed accounts:
- `VaultState`: version, bump, USDC mint, share mint, fixed allocation configuration, configured asset mints/programs, supported DEX program IDs, operation lock/sequence, accounting totals and immutable policy hash.
- `AssetVault`: PDA authority and token account for one configured asset.
- `DepositOperation`: depositor, isolated USDC escrow, per-asset staging accounts/amounts, completed step bitmap, actual spend/output, operation expiry, status and recovery rules. Staged deposit assets must remain outside the live vault balances until finalization.
- `RedemptionOperation`: owner, locked share amount, share-supply snapshot, proportional asset quantities, per-asset completion state, accumulated USDC proceeds, minimum USDC output, status and expiry.
- SPL share mint: standard SPL Token Program; no freeze authority; mint authority controlled by PDA.

Proposed instructions:
- `initialize_vault`
- `open_deposit`
- `execute_deposit_step` (permissionless)
- `finalize_deposit`
- `cancel_expired_deposit` / `refund_expired_deposit`
- `open_redemption`
- `execute_redemption_step` (permissionless)
- `finalize_redemption`
- `cancel_redemption` only before any asset is sold, or a carefully specified recovery path
- `close_operation`

No instruction may let a caller choose a different mint, arbitrary destination, DEX program, pool or swap route than the validated configuration allows.

## 7. Share accounting and oracle-free valuation

The protocol may use on-chain DEX pool state and executable quote calculations, but no external oracle.

The valuation module must:
- Use integer/fixed-point arithmetic only.
- Validate pool and market account ownership and expected layouts.
- Verify mint pair, token programs, pool state and route accounts.
- Enforce minimum liquidity, maximum price impact, maximum quote age/slot drift and per-asset exposure limits.
- Prefer independent routes where available; reject valuation if the configured confidence/liquidity checks cannot be met.
- Never trust caller-supplied prices.
- Apply conservative valuation haircuts or reject the operation where route depth is inadequate.
- Reject stale, manipulated, inconsistent or non-executable quotes.
- Use actual execution outputs and enforce minimum output for every swap.

Indicative NAV is:

`NAV_USDC = USDC_balance + sum(conservative_USDC_value(asset_i)) - liabilities`

For an ordinary deposit, shares are calculated against pre-deposit NAV and pre-deposit share supply. The design must serialize NAV-changing operations (or use an equivalent rigorously proven accounting method) so concurrent deposits/redemptions cannot use the same stale snapshot. A deposit snapshot expires after a bounded slot interval; expired requests must not mint shares using stale valuation.

For a non-empty vault:

`shares_to_mint = deposit_value_USDC * total_shares / pre_deposit_NAV_USDC`

For the first deposit, use a documented fixed initial share ratio (for example 1 share unit per 1 USDC base unit adjusted for decimals) to establish the initial accounting scale. All calculations use checked u128/u256 intermediate arithmetic as appropriate and explicit rounding rules. Dust and rounding residuals remain in the vault and are accounted for; they must not be silently assigned to a caller.

A critical implementation detail is that the staged deposit must not mint shares until every required allocation step has completed and the final contribution amount is known. The implementation must prove that pending assets cannot be counted both as existing NAV and as new contribution.

## 8. Deposit lifecycle

1. User opens a deposit with USDC and transaction-level slippage/expiry constraints.
2. Program escrows the USDC and snapshots pre-deposit NAV, share supply and slot.
3. The operation determines fixed target amounts from the portfolio weights. The USDC reserve portion remains USDC; other portions are swapped through configured routes.
4. Anyone may execute the next valid step. Each step validates the exact input/output mints, DEX program, pool, accounts, price impact and minimum output.
5. If any step cannot meet constraints, it does not advance. The operation may be retried within its validity window.
6. Once all steps complete, the program computes contribution value using actual balances/outputs, verifies snapshot freshness and accounting invariants, then mints shares.
7. If the operation expires before execution is complete, the program follows a deterministic recovery/refund path. Any assets already acquired must be handled by a documented unwind or claim-in-kind recovery mechanism; they cannot be silently confiscated or misaccounted.

The implementation must settle the exact behavior of partial execution and expiry before deposit code is considered complete.

## 9. Redemption lifecycle

1. User opens redemption for a share amount and minimum USDC output.
2. Program locks the shares in a PDA-controlled escrow; it does not burn them yet.
3. The operation snapshots the proportional fraction of each asset and reserves those quantities so other operations cannot spend them.
4. Anyone may execute each asset-to-USDC sale through approved DEX routes, with per-step minimum output and route validation.
5. After every required sale completes, the program verifies final USDC proceeds against the user's minimum.
6. It burns the locked shares and transfers one USDC payment.
7. If execution cannot complete, the request remains retryable. The program must define a safe expiry/recovery rule. Once any asset has been sold, cancellation cannot simply unlock the original shares without compensating for the sold assets; recovery must be designed explicitly.

Redemption uses the same share fraction for every holding. Fees and reserved liabilities must be included in the accounting model.

## 10. DEX integration and security boundary

The first implementation must choose specific supported DEX program(s) and audited CPI interfaces. Do not accept arbitrary CPI targets. For each adapter, validate program ID, pool account ownership, token mints, vault accounts, tick/bin/oracle accounts if required by that DEX, and all swap constraints.

DEX pool state is market data, not an oracle guarantee. Pool manipulation, low liquidity, sandwiching, stale accounts, malicious remaining accounts, token extensions and transfer fees are threat-model items. The contract must fail closed when the valuation adapter cannot establish a sufficiently conservative value.

## 10.1 Approved DEX routing architecture

**Primary recommendation: Jupiter Metis on-chain Router through a dedicated, narrowly scoped CPI adapter.** Use the Swap API V2 Router/build path only as an optional route-construction aid; it must not be a trusted price oracle, custody layer, transaction executor, or source of authorization. The Meta-Aggregator/RFQ managed-execution path is not used by the vault.

The route builder is replaceable: any frontend, keeper, or independent user may construct a candidate route. The vault program remains the enforcement boundary and must validate the exact input/output mints, source and destination token accounts, vault PDA authority, input amount, minimum output, allowed router program ID, allowed downstream DEX programs/pools, token program IDs, and all remaining accounts. No caller may redirect proceeds, add arbitrary fee recipients, substitute a mint, or invoke an unapproved program. The program records actual token balance deltas, not quote claims, as swap results.

The adapter must be implemented only after the precise deployed router program ID, CPI interface/IDL, account constraints, downstream-program behavior, upgrade authority, and supported Token-2022 behavior have been independently verified and pinned. If these checks cannot be satisfied, do not ship the Jupiter CPI adapter; use explicitly audited direct adapters instead.

**Fallback and decentralization boundary:** Keep the adapter interface modular so direct Raydium CPMM and Orca Whirlpool adapters can be added as independently audited alternatives. Do not add all adapters in the first release merely for breadth. First test whether the selected configured assets have adequate executable routes and liquidity through the primary router. A route-building API outage must not lock funds: permissionless clients can construct and submit valid routes using public on-chain state and the published instruction interface. No server-held key is required.

Jupiter's current documentation distinguishes the on-chain Metis Router/build path from the Meta-Aggregator path; the Router supports custom transaction composition/CPI and does not add a Jupiter swap fee, while the Meta-Aggregator includes additional routing engines and managed execution. The vault intentionally selects the Router path for program-controlled execution. This does not make every route risk-free: pool liquidity, downstream program upgrades, router upgrades, compute limits and CPI account-depth remain in the threat model.

## 10.2 Approved partial-redemption recovery: segregated recovery tickets

A USDC-only redemption cannot be guaranteed if a held token becomes permanently untradeable or no acceptable USDC route exists. No smart contract can manufacture USDC liquidity. The design must make this limitation explicit and provide a deterministic user-owned claim rather than pretending every withdrawal can always finish.

Recommended lifecycle:

1. **Preflight:** Before locking shares or selling anything, the client supplies candidate routes for every asset in the user's proportional basket. The program validates each route and its minimum output constraints. If any asset has no acceptable route, the redemption does not start; the user can wait or choose a different redemption amount/constraint.
2. **Lock and reserve:** Lock the requested shares in escrow and record the exact proportional quantities of each configured asset. Reserve those quantities so no other operation can spend them. Do not burn shares yet.
3. **Independent liquidation steps:** Each asset-to-USDC swap executes as its own permissionless step into the redemption's isolated USDC escrow. A failed CPI or min-output check reverts that step and leaves its state unchanged. Completed steps remain recorded and cannot be replayed or counted twice.
4. **Normal completion:** When every asset quantity has been liquidated, verify the total USDC against the user's minimum, burn the locked shares, and transfer one USDC payment.
5. **Recovery delay:** If the redemption remains incomplete beyond a fixed, documented delay (initial proposal: seven days), the owner may convert it into a **Recovery Ticket**. The conversion atomically moves all accumulated USDC and every unsold reserved token into ticket-specific escrow, burns the locked portfolio shares, marks the ticket claim as excluded from active vault NAV, and releases the global operation lock.
6. **Continue toward USDC:** Anyone may continue attempting approved asset-to-USDC swaps from the ticket escrow. When all residual assets have been sold, the ticket owner claims the accumulated USDC in one transfer.
7. **Explicit in-kind escape hatch:** If an asset is permanently untradeable, only the ticket owner may choose to claim the remaining token(s) in kind together with accumulated USDC. This is an explicit user action, not an automatic protocol substitution. The normal withdrawal path remains USDC-only.
8. **No confiscation or expiry forfeiture:** Tickets do not expire into protocol ownership. No administrator can redirect or seize ticket assets. A ticket can be closed only after its balances are fully claimed.

The Recovery Ticket is a PDA-backed claim record for version one, not necessarily a transferable token. It stores the owner, original share amount, remaining token balances, accumulated USDC, route/recovery status, and replay-protection sequence. A later transferable receipt can be considered separately after the claim-accounting model is audited.

**Accounting invariant:** Before conversion, locked shares remain in total share supply and redemption assets/proceeds remain attributable to the active redemption; the active NAV-changing operation is serialized. At conversion, the program atomically moves the entire residual claim out of active vault custody, burns the corresponding locked shares, and excludes ticket escrow balances from active NAV. This transition must be tested so assets are never counted both in vault NAV and as a ticket liability, and shares are never burned while the user loses the corresponding claim.

**Trade-off:** The ticket prevents a permanently untradeable asset from indefinitely blocking the main vault or trapping portfolio shares. It cannot guarantee an all-USDC outcome when liquidity does not exist. The user retains the exact residual asset claim and may keep waiting for liquidity or explicitly take it in kind.

## 11. Token compatibility and asset onboarding

The configured asset set is immutable after initialization in the initial design. The program accepts only exact configured mint addresses and supported token programs. Asset onboarding is a deployment prerequisite, not a permissionless arbitrary-mint feature.

Before an asset is configured, verify:
- Exact mint and token program.
- Issuer and legal/transfer restrictions.
- Token extensions and transfer behavior.
- Executable USDC route on supported DEX.
- Liquidity and slippage at realistic deposit/withdrawal sizes.
- Ability of the vault PDA to hold, transfer and sell the token.

A ticker or token metadata name is not sufficient verification. The global equity, emerging markets and individual-stock basket slots remain unresolved until exact assets are chosen. No placeholder mint may be deployed.

## 12. Security invariants

Tests must prove at minimum:
- Total allocation weights equal exactly 10,000 basis points.
- No unconfigured mint can enter or leave vault custody.
- Only the vault PDA can mint shares; no freeze authority exists.
- Shares minted equal the defined contribution/NAV formula with documented rounding.
- No shares mint before all deposit steps complete.
- No shares burn or USDC pay before all redemption steps complete.
- A redemption cannot reserve or sell more than the holder's pro-rata claim.
- Reserved balances cannot be spent by another operation.
- One stale NAV snapshot cannot be reused after expiry.
- A failed CPI leaves the current step and accounting unchanged.
- Slippage/minimum-output failures revert the step.
- All arithmetic is checked for overflow, underflow and division by zero.
- No caller can substitute a token mint, pool, DEX program or destination account.
- No administrator-only path can transfer vault assets or alter fixed weights.

## 13. Testing and delivery sequence

Use test-driven development:
1. Create Anchor workspace and document exact toolchain versions.
2. Write failing unit tests for allocation validation, fixed-point math, share calculations and rounding.
3. Implement pure accounting functions.
4. Add account/PDA constraints and initialization tests.
5. Add mocked/local-validator DEX adapter tests, then integration tests against pinned DEX program versions.
6. Test staged deposit and redemption state machines, partial completion, retries, expiry and recovery.
7. Add adversarial tests for stale/manipulated pool state, wrong accounts, unsupported token extensions, slippage and concurrent operations.
8. Run formatting, linting, unit tests and local-validator integration tests.
9. Obtain independent Solana program security review before any mainnet deployment.

No mainnet deployment or real-fund use is in scope for the initial implementation milestone.

## 13.1 Transaction-cost policy and deposit preview

The owner has accepted the following first-version policy:

- The wallet submitting a Solana transaction pays that transaction's network fee. When a permissionless keeper/third party submits an operation step, that submitter pays the network fee for that step.
- DEX swap fees, transfer fees where applicable, and execution price impact are reflected in the actual token amounts received. They are not separately billed by the program.
- The protocol charges no additional management, performance, or gas-reimbursement fee in version one.
- The vault does not operate a centralized gas sponsor or reimburse arbitrary transaction submitters.
- Any account-rent or token-account creation costs must be disclosed in the transaction flow and assigned explicitly; they must not be hidden in NAV accounting.

Before a deposit is finalized, the user interface should show an estimate of the net asset amounts expected to enter the portfolio after the planned swaps. The preview should itemize, per asset, the USDC allocation, estimated swap fee/price impact where the route exposes it, expected net token output, quote slot/time, and applicable slippage bounds. It must distinguish estimates from guaranteed outputs and show that final amounts can differ.

The on-chain program remains authoritative: it enforces configured routes, minimum outputs, expiry and accounting rules. The preview is informational and must never be trusted as an on-chain price input. If the final execution cannot satisfy the user's minimum-output constraints, the relevant step must fail rather than silently accept worse execution.

## 14. Open implementation decisions

These are engineering decisions to resolve and document before the relevant code is written:
- Exact DEX adapter(s) and pinned program versions.
- Exact verified mint addresses and token program types for all nine portfolio slots.
- Minimum liquidity and price-impact thresholds per asset.
- NAV route selection and conservative valuation algorithm.
- Initial share decimal/ratio and rounding policy.
- Deposit expiry and partial-execution recovery policy.
- Deposit expiry and partial-execution recovery policy.
- Confirmation that a single active NAV-changing operation at a time is acceptable for the first release (requests remain openable/queueable at any time, but execution is serialized).
- Exact seven-day Recovery Ticket clock semantics.
- Exact verified asset mints, Token-2022 compatibility, liquidity thresholds and NAV valuation methodology.

Approved architecture: Jupiter Metis Router as the primary adapter (subject to CPI verification), modular direct DEX adapter boundaries for later alternatives, and PDA-backed Recovery Tickets with an initial seven-day delay and owner-elected in-kind recovery. The default fee policy remains no protocol fee in v1; no upgrade authority after audit remains the deployment recommendation; serialized execution and explicit asset-specific limits remain recommended.
