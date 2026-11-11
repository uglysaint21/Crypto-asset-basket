# Solana Portfolio Vault — Security Remediation Decisions

**Status:** Recommended remediation baseline applied to the design and implementation plan  
**Date:** 2026-11-11  
**Branch:** design/solana-vault-architecture  
**Parent review:** docs/superpowers/reviews/2026-11-11-solana-portfolio-vault-architecture-security-review.md

## 1. Purpose and scope

This document converts the architecture review findings SEC-01 through SEC-11 into concrete design rules. It is authoritative where it is more specific than earlier prose. These are architecture/specification fixes, not implemented or tested contract code.

The main structural change is to remove the long-lived vault-wide operation lock. Deposits and redemptions use isolated escrow and per-operation state. Only short, atomic accounting transitions serialize naturally on Solana. Each operation has bounded lifetime and permissionless recovery.

## 2. Adopted security architecture

### 2.1 Replace the global long-lived lock with isolated operations

**Decision:** No operation may hold a vault-wide lock while waiting for swaps across multiple transactions.

- Every deposit and redemption has a unique PDA, monotonic operation ID, owner, status, creation timestamp, expiry timestamp, reserved quantities and operation-specific token accounts.
- Swaps execute only from that operation's escrow accounts.
- Deposits stage all purchased assets outside the live vault. Staging assets are excluded from live NAV and share supply until finalization.
- Redemption shares are locked in share escrow. The redemption reserves only the exact proportional quantities allocated to it. The locked shares remain in total SPL supply until normal completion or recovery-ticket conversion.
- Short state transitions—opening/reserving, finalizing, cancelling, or converting to a ticket—are atomic instructions. Solana transaction atomicity and account write locks serialize conflicting updates; no persistent global lock is held between transactions.
- Track aggregate locked-share amount, per-asset reserved amount, and operation counters in vault state. Every transition updates the operation PDA and aggregate counters atomically.
- A new redemption uses only unlocked shares and unreserved live holdings. Its share fraction is requested shares divided by total share supply less aggregate locked shares. Asset quantities are calculated from each live asset balance less already reserved quantities. Reject if requested shares exceed unlocked supply or any reservation exceeds available inventory.
- A deposit's authoritative NAV and share-supply snapshot is taken at finalization, not when the request is first opened. The staged contribution is valued and shares are calculated in the same finalization transaction. No stale opening-time NAV is used.
- For deposit share pricing, total NAV includes all assets and USDC still attributable to outstanding locked-share redemptions, including their isolated proceeds, until those shares are burned or converted into tickets. This keeps locked shares and their corresponding assets represented consistently.
- At redemption ticket conversion, all ticket balances leave active NAV and the locked shares are burned in the same atomic instruction. The ticket becomes a separate user claim and is excluded from active vault NAV.
- At normal redemption completion, shares are burned and USDC paid atomically. The redemption escrow is removed from active NAV at the same transition.
- Requests may be opened while other operations are executing, subject to available unlocked shares, unreserved inventory, and per-asset operation/account limits.

**Why:** An abandoned operation cannot freeze all users. Isolation also limits the amount of vault inventory exposed to any one DEX CPI.

### 2.2 Bounded operation lifetime and permissionless recovery

**Decision:** All multi-transaction operations have immutable deadlines that cannot be extended by retries.

- Initial operation lifetime: seven days from opened_at, recorded from Solana Clock timestamp. The timestamp is immutable.
- A deposit not finalized by expiry can be converted by anyone into a Deposit Recovery Ticket.
- A redemption with no successful liquidation step by expiry can be permissionlessly cancelled; locked shares return only to the recorded owner.
- A redemption with one or more successful liquidation steps by expiry can be permissionlessly converted into a Redemption Recovery Ticket.
- The caller who triggers cancellation or conversion receives no assets, shares, fees, or authority.
- A failed recovery transaction changes no balances or status and can be retried.
- Retries never reset opened_at, expires_at, or the ticket's original recovery terms.
- If a user wants a longer execution window, they must open a new operation after the old one has been safely resolved; an executor cannot silently extend the old one.

The previously approved seven-day recovery delay is therefore defined as seven days from operation opening, not seven days from the first successful swap. This gives every operation a finite maximum lock period and avoids an attacker delaying the first step indefinitely.

### 2.3 Deposit Recovery Tickets

**Decision:** A partially executed deposit never requires an assumed unwind route.

At expiry, anyone may atomically move the deposit's remaining USDC and all already-acquired staged tokens into a PDA-backed DepositRecoveryTicket. No portfolio shares are minted. The ticket records the original depositor as sole beneficiary, all token balances, USDC balance, per-asset execution floors, sequence/nonce and status.

- Anyone may continue approved swaps from ticket escrow to USDC, subject to depositor-approved execution floors.
- Only the depositor may claim USDC after all assets are converted.
- Only the depositor may explicitly elect to claim residual tokens in kind.
- The claim never expires into protocol ownership.
- Ticket balances are not part of active vault NAV.
- Closing is allowed only when every ticket balance is zero.
- The recovery caller cannot change the beneficiary, floors or destination.

This is the deposit-side counterpart to the approved Redemption Recovery Ticket.

### 2.4 Redemption Recovery Ticket minimum-output semantics

**Decision:** A Recovery Ticket preserves the original asset claim; it does not promise guaranteed USDC.

- Normal redemption finalization requires all assets liquidated and aggregate USDC proceeds at or above the original minimum total USDC.
- Conversion to a Recovery Ticket is a claim-preserving transfer, not a payout. It may happen after expiry even when accumulated USDC is below the original minimum, because every unsold asset and every USDC unit remains in the owner's ticket escrow.
- Every ticket asset stores an owner-approved minimum execution rate (fixed-point USDC base units per asset base unit) or equivalent amount floor. A permissionless executor cannot reduce it.
- For any partial asset sale, on-chain code derives the minimum output from the stored rate and actual input amount using checked arithmetic and conservative rounding.
- Only the ticket owner may update an execution floor. Each update increments a monotonic terms version and emits an event. Lowering a floor is explicit and irreversible for that version; the UI must show the old and new floor.
- The ticket owner may claim USDC when all assets are liquidated and accumulated USDC meets the original minimum.
- If final proceeds are below the original minimum, USDC claim requires an explicit owner-signed instruction accepting a lower total threshold. The instruction records the accepted threshold and emits an event.
- Alternatively, the owner may explicitly elect in-kind claim of remaining tokens with accumulated USDC. This is never automatic.
- An executor cannot redirect proceeds, select a beneficiary, or choose in-kind settlement.

### 2.5 Oracle-free NAV policy: no single-pool spot valuation

**Decision:** No single instantaneous DEX spot quote may determine share issuance.

For each asset that contributes to NAV:

1. Use on-chain time-weighted observations from supported pool implementations, not an external price feed.
2. Require at least two economically independent, validated liquidity venues where such venues exist. Two pools sharing the same underlying liquidity source do not count as independent.
3. Require a minimum observation history (initial baseline: 30 minutes) and reject immature, stale, discontinuous or inconsistent observations.
4. Compute a robust cross-venue reference (median of valid venue TWAPs), then compare it with a current executable vault-to-USDC sell quote at a configured notional size.
5. Reject valuation if spot execution deviates beyond the immutable per-asset deviation bound, if liquidity/price impact exceeds the configured threshold, or if venue observations disagree beyond the configured bound.
6. Apply a conservative valuation haircut and deterministic integer rounding only after all checks pass. The haircut, observation window, quote notional, liquidity floor and deviation bound are immutable per-asset configuration values selected from measured liquidity and attack-cost analysis; they must not be invented from ticker labels.
7. Never fall back to caller-supplied values, a single spot pool, or a cached off-chain quote.
8. If a safe valuation is unavailable, do not mint shares or finalize a deposit. Redemptions may continue using actual executable swap output and their user-defined minimums; if a route is unavailable, recovery-ticket rules apply.

**Important limitation:** A 30-minute TWAP and two pools reduce manipulation risk; they do not prove a market price is unmanipulable. Every enabled asset needs a documented manipulation-cost assessment at the maximum intended vault exposure. If an asset lacks suitable on-chain observation support or independent liquidity, it is not enabled for share issuance in v1.

The initial 30-minute observation window is a baseline to test, not a claim that 30 minutes is universally sufficient. Per-asset thresholds remain a pre-implementation configuration gate.

### 2.6 First-deposit, share inflation and donation policy

**Decision:** Use virtual accounting offsets, minimum deposit limits and an internal recognized-balance ledger.

- Define immutable virtual asset and virtual share offsets, plus a minimum initial deposit, before implementing share math. Select values based on share decimals and precision analysis; do not use an arbitrary example ratio.
- Use the offsets in the share conversion formula for both initial and subsequent deposits, with checked wide-integer arithmetic and conservative rounding.
- Reject a deposit if calculated shares are zero or below the configured minimum.
- Reject a redemption if calculated asset quantities are all zero or if it would violate minimum residual/dust rules.
- Maintain per-asset accounted_balance and per-operation reserved/staged balances. A direct unsolicited transfer that causes actual token balance to exceed the recognized ledger is classified as unrecognized surplus and is not counted in NAV or credited to a depositor.
- Every program-controlled transfer/swap updates the ledger from actual pre/post token balance deltas. Before spending, require actual balance to be at least the recognized balance.
- Unrecognized surplus is not sweepable by an administrator. It remains segregated from shareholder accounting; a future permissionless donation-adoption mechanism would require separate design and approval.
- Round share issuance down; round required asset reservations conservatively so the protocol cannot promise more inventory than exists. All dust treatment is deterministic and tested.

### 2.7 Token program and extension admission policy

**Decision:** V1 accepts only explicitly verified mint/program combinations and rejects unsupported extensions by default.

- The default v1 policy is the legacy SPL Token Program for portfolio assets and the configured USDC mint/program. Token-2022 assets are rejected until their extensions have been individually reviewed and implemented.
- If Token-2022 support is later proposed, explicitly inspect and test transfer-fee, transfer-hook, permanent-delegate, pausable, default-account-state, confidential-transfer, CPI-guard and other present/future extensions.
- Reject non-transferable, confidential-transfer, permanent-delegate, transfer-hook, pausable/default-frozen or otherwise unsupported extension combinations unless a later reviewed specification explicitly allows them.
- Require immutable/revoked mint authority and freeze authority by default for portfolio assets. If an intended tokenized asset requires an active issuer authority, it is an explicit issuer-risk exception and must be documented and accepted before that mint is configured. The program cannot prevent an issuer from freezing or otherwise restricting a token where the issuer retains that power.
- Inventory all mint/freeze/extension authorities and issuer restrictions off-chain as part of asset onboarding. A ticker or website claim is not evidence of transferability or backing.
- Use actual received balance deltas for any specifically approved transfer-fee token; do not assume requested amount equals received amount.

This stricter default may disqualify some tokenized ETF/stock candidates. Do not weaken it silently to preserve a ticker list.

### 2.8 Jupiter Router CPI isolation and verification gate

**Decision:** Jupiter remains the primary candidate, but no CPI adapter is implemented until the exact program interface can be verified and constrained.

- Treat /build as an off-chain raw-instruction construction endpoint, not an on-chain program and not a trusted quote/oracle.
- Pin exact target cluster, program ID, deployed binary/source or reproducible build, IDL/instruction discriminator, account ordering/privileges, upgrade authority, and supported downstream AMM program IDs.
- Accept only the verified swap instruction variant. Reject setup, cleanup, compute-budget, tip, arbitrary auxiliary instructions and unknown discriminators from the CPI path.
- Parse and validate every account meta and relevant instruction field before invocation.
- Use operation-specific source and destination token accounts. Never expose the shared live vault token account to a router CPI.
- Transfer only the exact step input amount into the operation escrow; invoke the adapter; verify source debit and destination credit from actual balance deltas; reject any unexpected debit, destination, mint, authority or residual balance movement.
- The route builder may be unavailable; independent clients must be able to construct candidate instructions. The program validates them on-chain.
- If the verified Router program cannot meet these restrictions, stop. Do not grant it access to shared vault balances. Bring back a direct DEX adapter alternative for a separate review/approval rather than weakening validation.

The official Jupiter documentation distinguishes Router (/build + /submit, Metis on-chain routing and raw instructions for CPI/custom composition) from Meta-Aggregator (/order + /execute, managed execution and other routing engines). This project selects the Router path for composability, but its exact deployed CPI contract still requires verification.

### 2.9 Queueing and reservation invariants

- A queued request contains intent and limits only; it does not snapshot NAV or reserve balances until activation.
- Activation occurs atomically and uses then-current share supply and available recognized balances.
- Redemption opening locks shares and reserves proportional quantities from unreserved live inventory. Aggregate locked shares and per-asset reserved quantities are maintained in vault state.
- Deposits do not reserve live portfolio inventory. They use isolated staging accounts.
- Pending redemption assets/proceeds remain attributed to locked shares and are included in total NAV while those shares remain in total supply.
- At normal redemption settlement or ticket conversion, assets/proceeds leave active NAV exactly when the associated shares are burned.
- Deposit staging and all Recovery Ticket balances are excluded from active NAV.
- Every operation and ticket has unique ID, status, sequence and terms version. Replays, stale sequences and duplicate account aliases are rejected.

### 2.10 Upgrade authority and immutability

- During development/testnet, retain upgrade authority only in a secure development multisig, never a single hot wallet.
- Before mainnet, complete independent audit, reproducible build verification, deployment rehearsal and a documented incident/migration plan.
- The preferred mainnet posture is revoke upgrade authority after those gates. This makes the deployed code immutable and removes upgrade-key custody risk, but also means critical bugs cannot be patched in place.
- If the owner chooses upgradeability instead, that is an explicit trust trade-off requiring a public multisig/timelock governance design and must not be described as fully trustless.
- No admin authority may withdraw vault assets, change user claims, seize tickets or alter asset weights under either posture.

### 2.11 Portfolio-weight semantics

The configured percentages are new-deposit allocation weights, not a promise that actual portfolio holdings remain at those percentages. Relative price changes will cause drift because automatic rebalancing is not part of v1. User-facing reporting must show actual weights and drift. Any future underweight-directed deposit allocation or rebalancing is a separate strategy change requiring approval.

### 2.12 Compatibility-driven portfolio rebalance and asset admission

The prior proposed investment basket is superseded for v1. A review of issuer documentation and current Solana token standards found that the named tokenized securities/commodities are incompatible with the strict v1 admission policy or lack a verified mint/authority record.

- SPYx and QQQx: xStocks' Solana architecture uses Token-2022. Its corporate-action implementation relies on Token-2022 scaled-UI functionality; public token records show issuer-controlled extension/authority risks. Exclude under legacy-SPL-only policy.
- PAXG on Solana: Paxos explicitly states the Solana PAXG deployment uses Token-2022 and a Permanent Delegate. Exclude.
- COPX and USOon: exact mint/program/authority state was not established in the asset register. If treated as Ondo Global Markets assets, the published Ondo Solana program uses Token-2022 with Pausable, ConfidentialTransferMint, TransferHook and DefaultAccountState extensions. Exclude.
- Global equity, emerging-markets and individual-stock basket slots: placeholders without exact selected mints and authority evidence. Remove from allocation until individually verified.

**Interim weights:** 100% native USDC, 0% non-USDC investment assets. This is a defensive, temporary cash-only configuration, not a diversified portfolio. The USDC mint is legacy SPL, but its mint and freeze authorities are active and Circle retains blocklisting/freeze powers. It is an explicit settlement-asset exception only; the protocol cannot prevent Circle from freezing USDC or guarantee that a frozen vault account can transfer or pay redemptions.

**Admission gate:** exact mint and token program allowlist; on-chain authority/extension validation; legacy SPL-only for non-USDC v1 assets; issuer/custody/legal restrictions documented; two independent on-chain TWAP venues; executable vault-sized sell route; measured liquidity/manipulation-cost thresholds; exact buy/sell and recovery tests. A ticker or DEX listing is never sufficient. No silent replacement or admin hot-add.

**Candidate, not approved:** wrapped SOL may be assessed as a native-asset candidate under the legacy Token Program. JitoSOL and mSOL require additional authority and liquid-staking risk review; they are not approved merely because they are standard SPL tokens.

**Additional controls:** immutable exact-mint registry/configuration hash; revalidation of token program and mint account at transfer/valuation boundaries; per-asset exposure and operation-size caps; deterministic fail-closed circuit breakers for stale/divergent valuation or insufficient executable liquidity; existing user claims remain recoverable and are never confiscated. USDC issuer risk must be disclosed in the UI and deployment documentation.

## 3. Finding disposition

| Finding | Disposition | Remediation |
|---|---|---|
| SEC-01 global lock griefing | Addressed in design | No long-lived global lock; isolated operations; seven-day immutable expiry; permissionless cancel/ticket conversion |
| SEC-02 manipulated NAV | Mitigated with fail-closed policy; asset-specific verification remains | Multi-venue on-chain TWAP, executable quote deviation guard, haircut, no single-pool fallback |
| SEC-03 Jupiter CPI trust boundary | Verification blocker remains | Exact program/interface pinning, strict instruction/account validation, operation escrow isolation; no integration if unverifiable |
| SEC-04 ticket poor execution | Addressed in design | Owner-set per-asset execution floors; only owner can lower them |
| SEC-05 minimum-USDC ambiguity | Addressed in design | Ticket conversion preserves full asset claim; below-minimum USDC claim requires explicit owner consent |
| SEC-06 partial deposit stuck funds | Addressed in design | Permissionless Deposit Recovery Ticket with USDC/in-kind owner claim |
| SEC-07 first-deposit/donation attacks | Addressed in design; constants need analysis | Virtual offsets, minimum deposit/share limits, recognized-balance ledger, unrecognized surplus excluded |
| SEC-08 issuer/Token-2022 risks | Mitigated by strict v1 admission policy | Legacy SPL-only default; reject unsupported extensions and mutable authorities unless explicit reviewed exception |
| SEC-09 queue/reservation double counting | Addressed in design | Activation-time snapshots, locked-share/reserved-inventory ledgers, consistent NAV treatment |
| SEC-10 upgrade authority | Deployment decision retained | Development multisig; immutable mainnet preferred after audit and rehearsal |
| SEC-11 target-weight drift | Addressed as product semantics | Weights explicitly apply to new deposits; show actual drift; no implied rebalancing |

## 4. Remaining release blockers (not papered over)

The following cannot honestly be marked fixed by prose alone:

1. NAV support per asset: verify two independent on-chain observation venues, observation history, liquidity and attack cost. Assets failing this gate are disabled for share issuance.
2. Jupiter CPI: verify the exact deployment, instruction, account semantics, downstream route behavior and upgrade authority on the selected cluster. If impossible, use a reviewed direct adapter instead.
3. Asset mints and issuer authorities: verify exact mint addresses, token program, mint/freeze authorities, extension state, issuer restrictions and executable liquidity.
4. Numeric risk parameters: set per-asset TWAP window, liquidity floor, max price impact, max deviation, valuation haircut, min execution rate and max operation amount from measured data and attack-cost analysis.
5. Virtual accounting constants: choose share decimals, virtual asset/share offsets, minimum initial deposit, minimum share mint and dust rules after formal arithmetic analysis.
6. Independent audit: architecture and tests are not a substitute for an independent Solana program audit before mainnet.

## 5. Verification required during implementation

The test suite must include:
- State-machine property tests over arbitrary operation interleavings.
- Abandoned deposit/redemption operations and permissionless recovery.
- Concurrent redemptions proving locked shares and reserved assets cannot be double-allocated.
- Deposits finalized while redemptions are pending, proving NAV includes locked-share claims consistently.
- Ticket conversion and claim conservation, including below-minimum proceeds and owner floor updates.
- Manipulated pool spot prices, stale/immature TWAPs, correlated venues and missing venue observations.
- First-deposit donation/inflation simulations, unsolicited transfers and long-run rounding.
- Wrong Jupiter instruction, program ID, account order, signer, mint, source/destination, downstream program and unexpected balance-delta tests.
- Unsupported token program/extensions and mutable issuer authority tests.
- Upgrade authority verification and reproducible build checks.

**Scope statement:** This document changes the written architecture and implementation requirements only. No smart-contract source code has been implemented, and no tests have been run as part of this remediation.


## 6. Technical references

- Jupiter Swap API V2 overview: https://developers.jup.ag/docs/swap — documents the Router (/build and /submit) versus Meta-Aggregator (/order and /execute) distinction, Metis routing and raw instruction composition.
- Jupiter custom swap guide: https://developers.jup.ag/docs/guides/how-to-build-a-custom-swap-with-metis — useful historical implementation context; current integrations should follow the V2 docs where they supersede older guidance.
- Solana CPI documentation: https://solana.com/docs/core/cpi — PDA signing, account privilege propagation and CPI limits.
- Solana PDA derivation: https://solana.com/docs/core/pda/pda-derivation — canonical PDA derivation and bump validation.
- Solana security guidance: https://solana.com/docs/tools/security — account/program validation and security practices.
- Token-2022 documentation: https://www.solana-program.com/docs/token-2022 — extension model and token compatibility considerations.
