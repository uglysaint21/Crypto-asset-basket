# Solana Portfolio Vault — Design Specification

**Status:** Architecture and security-remediation baseline accepted; implementation subject to release gates  
**Date:** 2026-11-11  
**Repository:** `uglysaint21/Crypto-asset-basket`

## 1. Intent

Build a non-custodial, on-chain Solana portfolio vault. Users deposit USDC at any time. The program invests deposits according to fixed target weights, holds the resulting tokenized assets in PDA-controlled token accounts, and issues a transferable SPL portfolio share token. A user may redeem shares for one USDC payment; the vault liquidates the corresponding proportional quantities of its holdings through supported Solana DEX programs. The seven-day operation deadline is immutable from opening; incomplete operations follow permissionless recovery rules.

The protocol must not depend on external price oracles, a hosted investment server, an administrator who can move assets, or an administrator who can change investment policy.

## 2. Decisions supplied by the project owner

### Target portfolio weights

| Slot | Intended exposure | Weight | Admission state |
|---|---|---:|---|
| 0 | Native USDC settlement/reserve (legacy SPL Token Program) | **100%** | **Interim configuration only**; exact mint and live authority state must be verified at deployment |
| | **Total** | **100%** | No portfolio investment asset is currently approved |

**Compatibility-driven rebalance (2026-11-11):** The previous 85% allocation to non-USDC exposures is removed from the v1 configuration. The only interim configured holding is native Solana USDC at 100%. This is a defensive cash-only configuration, not a diversified investment portfolio and not a claim that USDC is risk-free. It remains subject to Circle issuer/blocklist/freeze risk and is an explicit, narrowly scoped exception to the default revoked-authority rule because USDC is the required deposit, settlement and reserve asset. No security, ETF, commodity or staking token is silently substituted. The portfolio remains in asset-onboarding mode until at least one non-USDC asset passes every admission gate in section 14.7.

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
- Aggregate locked-share and per-asset reserved-balance accounting; no long-lived vault-wide operation lock.

DEX integrations are direct Solana CPIs through audited adapters. The first adapter(s) must be explicitly selected and pinned to known program IDs and account layouts. A hosted aggregator is not a mandatory dependency. A frontend or indexer may submit transactions and display events but is never authoritative for balances or permissions.

## 6. State and instruction boundaries

Proposed accounts:
- `VaultState`: version, bump, USDC mint, share mint, fixed allocation configuration, configured asset mints/programs, supported DEX program IDs, operation sequence, aggregate locked shares, per-asset reserved balances, recognized balances and immutable policy hash.
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

For an ordinary deposit, shares are calculated against pre-deposit NAV and pre-deposit share supply. The design must serialize only short atomic accounting transitions, not hold a vault-wide lock between transactions. Deposits take their authoritative NAV/share-supply snapshot at finalization. Redemptions atomically lock shares and reserve only unreserved inventory. Pending redemption assets/proceeds remain represented in NAV while their shares remain in supply; they leave active NAV atomically when shares are burned or converted into a ticket. See section 14 for the complete reservation model.

For a non-empty vault:

`shares_to_mint = deposit_value_USDC * total_shares / pre_deposit_NAV_USDC`

For the first deposit, use a documented fixed initial share ratio (for example 1 share unit per 1 USDC base unit adjusted for decimals) to establish the initial accounting scale. All calculations use checked u128/u256 intermediate arithmetic as appropriate and explicit rounding rules. Dust and rounding residuals remain in the vault and are accounted for; they must not be silently assigned to a caller.

A critical implementation detail is that the staged deposit must not mint shares until every required allocation step has completed and the final contribution amount is known. The implementation must prove that pending assets cannot be counted both as existing NAV and as new contribution.

## 8. Deposit lifecycle

1. User opens a deposit with USDC and transaction-level slippage/expiry constraints.
2. Program escrows USDC in operation-specific accounts. The authoritative NAV and share-supply snapshot is taken at finalization, not at request opening.
3. The operation determines fixed target amounts from the portfolio weights. The USDC reserve portion remains USDC; other portions are swapped through configured routes.
4. Anyone may execute the next valid step. Each step validates the exact input/output mints, DEX program, pool, accounts, price impact and minimum output.
5. If any step cannot meet constraints, it does not advance. The operation may be retried within its validity window.
6. Once all steps complete, the program computes contribution value using actual balances/outputs, verifies snapshot freshness and accounting invariants, then mints shares.
7. If the operation expires before execution is complete, anyone may convert it into a Deposit Recovery Ticket containing all residual USDC and acquired tokens. The depositor remains the sole beneficiary; permissionless swaps are subject to depositor-approved floors, and the depositor may explicitly claim residual assets in kind.

Partial-deposit expiry is resolved by the Deposit Recovery Ticket rules in section 14.3; the implementation must test claim conservation for every partially completed state.

## 9. Redemption lifecycle

1. User opens redemption for a share amount and minimum USDC output.
2. Program locks the shares in a PDA-controlled escrow; it does not burn them yet.
3. The operation snapshots the proportional fraction of each asset and reserves those quantities so other operations cannot spend them.
4. Anyone may execute each asset-to-USDC sale through approved DEX routes, with per-step minimum output and route validation.
5. After every required sale completes, the program verifies final USDC proceeds against the user's minimum.
6. It burns the locked shares and transfers one USDC payment.
7. Every redemption expires seven days after opening. If no asset sale has succeeded, anyone may cancel after expiry and unlock shares only to the recorded owner. If any sale has succeeded, anyone may convert the redemption to its owner's Recovery Ticket after expiry. Conversion moves every remaining token and accumulated USDC into ticket escrow and burns locked shares atomically.

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

The Recovery Ticket is a PDA-backed claim record for version one, not necessarily a transferable token. It stores the owner, original share amount, remaining token balances, accumulated USDC, route/recovery status, owner-set per-asset minimum execution rates, terms version and replay-protection sequence. Anyone may trigger conversion after expiry; only the owner may change floors or claim proceeds/assets. A later transferable receipt can be considered separately after the claim-accounting model is audited.

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
- Confirmation of per-asset maximum concurrent operation and reservation limits; no long-lived global operation lock is used.
- Seven-day expiry is fixed from operation opening and cannot be extended by retries.
- Exact verified asset mints, Token-2022 compatibility, liquidity thresholds and NAV valuation methodology.

Approved architecture: Jupiter Metis Router as the primary adapter (subject to CPI verification), modular direct DEX adapter boundaries for later alternatives, and PDA-backed Recovery Tickets with an initial seven-day delay and owner-elected in-kind recovery. The default fee policy remains no protocol fee in v1; no upgrade authority after audit remains the deployment recommendation; isolated operation execution, short atomic accounting transitions and explicit asset-specific limits are the security baseline.


## 14. Security remediation baseline (authoritative)

The full remediation decisions are recorded in [the security remediation document](../reviews/2026-11-11-solana-portfolio-vault-security-remediation.md). This section makes the core rules part of the design specification and overrides any older conflicting wording above.

### 14.1 No long-lived global operation lock

Deposits and redemptions use isolated operation PDAs and escrow accounts. No operation may hold a vault-wide lock while waiting across transactions. Short accounting transitions are atomic. Aggregate locked shares and per-asset reserved quantities prevent overlapping redemptions. Queued requests take no NAV snapshot and reserve no balances until activation.

For a new redemption, use unlocked share supply (total supply less already locked shares) and unreserved live inventory. Pending redemption assets and proceeds remain attributed to locked shares and included in total NAV until those shares are burned. At normal settlement or ticket conversion, assets leave active NAV atomically with share burn.

### 14.2 Immutable seven-day expiry and permissionless recovery

Every multi-transaction deposit/redemption has an immutable seven-day deadline from its opening timestamp. Retries cannot extend it. After expiry:
- An unstarted redemption can be cancelled by anyone; shares return only to the recorded owner.
- A partially executed redemption can be converted by anyone into its owner's Recovery Ticket.
- An incomplete deposit can be converted by anyone into a Deposit Recovery Ticket.

The caller receives no custody, claim or authority. Recovery failure is retryable and must not mutate balances.

### 14.3 Deposit Recovery Ticket

A partially executed deposit transfers all remaining USDC and acquired staged assets into a PDA-backed ticket. No shares are minted. The depositor is the sole beneficiary. Anyone may continue swaps subject to owner-set execution floors; only the depositor may claim USDC after full conversion or explicitly elect in-kind claim. Tickets never expire into protocol ownership and remain excluded from active NAV.

### 14.4 Ticket execution and minimum-USDC protection

Each ticket asset has an owner-approved minimum execution rate. Permissionless executors cannot lower it. Only the owner may update it, with a monotonic terms version and event. Normal redemption requires aggregate USDC at or above the original minimum. Ticket conversion preserves the full residual claim and is not a payout. If all assets are sold below the original minimum, USDC claim requires explicit owner consent to a lower threshold; otherwise the owner may elect in-kind recovery where assets remain.

### 14.5 Manipulation-resistant oracle-free valuation

No single instantaneous pool spot quote may determine NAV or share issuance. Each enabled asset must have a documented on-chain observation method, initially targeting at least two economically independent liquidity venues and a 30-minute minimum observation history, plus a current executable sell quote, cross-venue deviation checks, liquidity/price-impact bounds and a conservative haircut. These are initial baselines, not universal guarantees; numeric limits are immutable per asset and must be justified by measured liquidity and attack-cost analysis.

If reliable observations or sufficient independent liquidity are unavailable, deposits cannot finalize or mint shares. Do not fall back to caller prices, off-chain quotes or a single spot pool. Redemptions can continue using actual execution and user minimums where routes exist.

### 14.6 Share math, recognized balances and unsolicited transfers

Use immutable virtual asset/share offsets, a minimum initial deposit, minimum share-mint rules, checked wide-integer math and conservative rounding. The exact constants must be derived and tested before implementation. Maintain a recognized-balance ledger. Unsolicited transfers above the ledger are unrecognized surplus and are excluded from NAV; no administrator may sweep them. All program-controlled movements update the ledger from actual balance deltas.

### 14.7 V1 token compatibility

V1 defaults to the legacy SPL Token Program for portfolio assets. Token-2022 mints are rejected until extensions are individually reviewed and supported. Require revoked mint and freeze authorities by default. Any exception for an issuer-controlled authority must be documented and explicitly accepted before that asset is enabled. Unsupported extensions, transfer hooks, permanent delegates, pausable/default-frozen states and non-transferable/confidential assets are rejected.

### 14.8 Jupiter CPI isolation and fail-closed gate

Jupiter Swap API /build is an off-chain instruction builder, not an on-chain program or trusted oracle. Before implementing CPI, pin and verify the exact deployed program ID, instruction discriminator/data, account order/privileges, downstream program behavior, source/IDL revision and upgrade authority. Only the verified swap instruction is accepted. Operation-specific escrow accounts are exposed to CPI; shared live vault token accounts are never passed to the router. Validate all mints, token programs, source/destination accounts, PDA signer, amount, minimum output, remaining accounts and actual balance deltas. If these constraints cannot be proven, do not implement the Jupiter adapter; return with a direct adapter alternative for review.

### 14.9 Upgrade and portfolio-weight semantics

Development upgrade authority is held by a secure multisig. Mainnet immutability after independent audit, reproducible build verification and deployment rehearsal is preferred; retaining upgradeability is an explicit trust trade-off. The configured percentages describe new-deposit allocation weights, not continuously maintained portfolio weights. Actual weight drift must be displayed; no automatic rebalancing is implied.

### 14.11 Compatibility-driven asset admission and interim rebalance

**Interim portfolio:** 100% native USDC (Solana legacy SPL Token Program). This is a temporary defensive configuration, not the intended long-term diversified portfolio. No non-USDC portfolio asset is approved for v1 at this review point. Do not treat this as investment advice or as a promise of principal safety.

**Removed from the previous target allocation:**
- SPYx and QQQx: Solana xStocks uses Token-2022 for its corporate-action mechanism; issuer-controlled extensions/authorities conflict with the current v1 legacy-SPL-only and no-permanent-delegate admission posture.
- PAXG on Solana: Paxos confirms that the Solana deployment uses Token-2022 and a Permanent Delegate. This is expressly incompatible with v1.
- COPX and USOon: exact Solana mints and current extension/authority records were never established in the approved asset register. If considered as Ondo Global Markets assets, the issuer's published Solana implementation uses Token-2022 and includes Pausable, ConfidentialTransferMint, TransferHook and DefaultAccountState extensions. They are not admitted.
- Global-equity ETF, emerging-markets ETF and individual tokenized-stock basket: these were unselected placeholders, not verified mint configurations. They are removed rather than left as implicit slots. xStocks or Ondo securities must not be reintroduced by ticker alone.

These are v1 compatibility decisions, not assertions that the issuers' products are inherently unsafe. The protocol deliberately declines their issuer-control and Token-2022 feature surface under the present threat model.

**USDC exception and issuer risk:** Solana USDC is a legacy SPL token, but its mint authority and freeze authority are active. Circle's terms also reserve blocklisting/freeze powers. The protocol accepts this only because USDC is the required base settlement asset; it is not an approved general-purpose investment holding under the revoked-authority rule. Document the exact mint, Token Program ID, decimals and observed authorities in the deployment manifest. A Circle freeze/blocklist action can make deposits, swaps, redemption payments or claims involving a frozen vault account fail. No vault program can override the issuer. Never describe the resulting vault as censorship-resistant with respect to USDC.

**Future admission gate (all conditions required):**
1. Hardcode/allowlist the exact mainnet mint and exact token program ID. Reject symbol/name/metadata matching and caller-selected mints.
2. Decode and validate the mint account on-chain; record decimals, mint authority, freeze authority, token program and any extension state in an immutable asset manifest/configuration hash.
3. V1 non-USDC assets must use the legacy SPL Token Program, with mint and freeze authorities revoked, unless a specific, separately reviewed issuer-risk exception is approved. No silent Token-2022 acceptance.
4. Require no permanent delegate, transfer hook, pausable, default-frozen, confidential-transfer, transfer-fee or other unsupported extension. For any future Token-2022 proposal, require a separate threat model, adapter tests and explicit owner approval before changing this policy.
5. Require documented issuer/custody/redemption terms and jurisdiction/transfer restrictions; on-chain token compatibility does not establish legal ownership, reserve backing or redemption rights.
6. Require at least two economically independent on-chain TWAP venues, minimum observation history, executable vault-sized sell quotes, liquidity/impact bounds, and measured manipulation-cost analysis. Two pools sharing the same underlying liquidity source do not count.
7. Require a successful exact-mint buy and sell route test, including actual balance deltas, minimum-output enforcement, transfer behavior and recovery behavior. Record the date, slot and evidence.
8. Set per-asset maximum exposure, maximum operation size, minimum liquidity, deviation bound and haircut from measured market data and attack-cost analysis. Do not invent universal numeric values.
9. A token that fails any gate is excluded from share-issuance NAV and from deposit allocation. No fallback to caller-supplied prices or alternate mints.

**Candidate research is not approval:** Wrapped SOL may be evaluated as a native-asset candidate because Solana documents its native mint under the legacy Token Program. Liquid-staking tokens such as JitoSOL and mSOL are not automatically approved: their stake-pool minting/redemption authority and depeg/stake-pool risks need separate review, and the current authority policy may disqualify them. No replacement weight is assigned until that review is complete.

**No automatic substitution:** If all non-USDC candidates fail, retain the 100% USDC interim configuration or pause launch of the investment product. Do not automatically fill vacant weights with SOL, another stablecoin, a wrapped asset or an issuer-controlled security token.

### 14.12 Additional fail-closed controls

- The on-chain asset registry is an exact-mint allowlist; asset identity never comes from ticker, symbol, metadata URI or an off-chain API.
- Validate the configured token program and mint account owner on every instruction boundary that transfers or values an asset. Reject account substitution and duplicate mint aliases.
- Bind each asset's route allowlist, valuation adapter, thresholds and maximum exposure into the immutable policy hash. No administrator can hot-add a mint or change weights.
- Enforce per-asset exposure and per-operation amount caps selected from the liquidity/manipulation analysis. Cap calculations use recognized balances and reserved balances, not raw token-account balances.
- Add deterministic per-asset circuit breakers: if required observations become stale, venues diverge, executable liquidity falls below its configured floor, or token-account/mint invariants fail, block new deposits and purchases involving that asset. Do not trap existing claims: allow safe redemption/recovery paths to continue where executable, otherwise preserve the owner's recovery ticket.
- Do not add a discretionary admin pause that can seize, freeze or redirect user claims. Any emergency stop semantics must be narrowly scoped, deterministic, auditable and unable to transfer custody.
- Treat USDC issuer freeze/blocklist as an external dependency. Expose this risk in the user interface and deployment checklist; do not claim the vault can guarantee settlement if the issuer freezes its token accounts.

### 14.10 Release blockers

No asset is enabled until its exact mint, token program, issuer authorities, extension state, on-chain observation support, independent liquidity venues, executable route and per-asset risk parameters are verified. Virtual share offsets and minimums must be mathematically selected. The exact Jupiter CPI contract must be verified before integration. These are implementation gates, not assumptions that can be resolved by documentation alone.
