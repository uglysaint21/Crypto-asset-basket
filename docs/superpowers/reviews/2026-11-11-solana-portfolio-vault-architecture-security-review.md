# Solana Portfolio Vault — Architecture Security Review

**Review type:** Architecture / threat-model review (not a source-code audit)  
**Date:** 2026-11-11  
**Repository:** `uglysaint21/Crypto-asset-basket`  
**Branch:** `design/solana-vault-architecture`  
**Reviewed documents:**
- `docs/superpowers/specs/2026-11-11-solana-portfolio-vault-design.md`
- `docs/superpowers/plans/2026-11-11-solana-portfolio-vault-implementation-plan.md`

## 1. Executive verdict

**Do not begin production-oriented contract implementation until the Critical/High design findings below are resolved in the specification and translated into tests.**

The architecture has good foundations: fixed asset configuration, no arbitrary CPI target, PDA-controlled custody, staged operations, actual token balance-delta accounting, no external oracle dependency, and a user-owned recovery claim.

However, the current documents leave several important security rules as open decisions or imply protections that are not yet specified precisely enough. The biggest risks are:

1. A long-lived global operation lock can be abused to block the entire vault.
2. Oracle-free valuation can still be manipulated through on-chain liquidity pools.
3. The Jupiter integration trust boundary and exact CPI contract are not yet established.
4. Permissionless recovery-ticket execution can expose users to poor execution unless the ticket itself stores enforceable price floors.
5. Redemption minimum-output semantics become ambiguous when a ticket is converted.
6. Partial deposit recovery is not designed yet.
7. First-deposit/share rounding and unsolicited token donations need explicit accounting protections.

These are design risks identified from the written architecture. No deployed program or source code was available in this review, so this is not a claim that an exploit exists in code or that an audit has been completed.

## 2. Findings by severity

### SEC-01 — Global operation lock can be griefed into a vault-wide denial of service

**Severity: Critical design blocker**

**Affected design:** One active NAV-changing operation at a time; permissionless staged execution; redemption recovery after seven days.

**Attack scenario**
1. An attacker opens a deposit or redemption that acquires the global operation lock.
2. The attacker deliberately does not execute the next step.
3. The operation remains pending and blocks other deposits/redemptions.
4. If recovery requires the operation owner to trigger it, the attacker can simply remain inactive indefinitely.

The seven-day ticket delay does not solve this if the transition is owner-only. A permissionless actor must be able to make the state progress while preserving the original owner's claim.

**Required changes**
- Define a finite operation deadline at creation.
- If no redemption asset has been sold, allow anyone to abort the operation after expiry and return the locked shares to the recorded owner.
- If any asset has been sold, allow anyone to trigger Recovery Ticket conversion after the recovery deadline. The ticket beneficiary remains the original owner; the caller receives no claim or authority.
- Define a permissionless expiry/refund/recovery path for partially completed deposits as well.
- Do not let retries reset the operation's original deadline.
- Prefer a short NAV-settlement critical section rather than holding a vault-wide lock throughout a multi-day swap sequence. If v1 retains a global lock, enforce hard deadlines and permissionless recovery for every operation state.

**Tests required**
- Attacker opens and abandons each operation state; another user can resume vault activity after the documented deadline.
- Non-owner can trigger expiry or ticket conversion but cannot redirect any shares, USDC or residual tokens.
- Repeated retries cannot extend the original deadline.
- A failed expiry/recovery transaction leaves state unchanged and remains retryable.

### SEC-02 — On-chain DEX valuation is manipulable unless a concrete observation model is specified

**Severity: Critical design blocker**

**Affected design:** NAV based on on-chain DEX pool state and executable quotes, with no external oracle.

**Attack scenario**
1. An attacker uses a flash-funded or otherwise temporary trade to move a thin pool's spot price.
2. A deposit or redemption uses that manipulated pool state to value the existing portfolio.
3. The program issues too many or too few shares, or accepts an unfair redemption value.
4. The attacker unwinds the manipulation after the accounting operation.

A DEX quote is evidence of an executable route at a particular moment; it is not automatically a manipulation-resistant NAV oracle. A minimum liquidity check and a caller-supplied minimum output do not, by themselves, prevent a temporary price distortion.

**Required changes**
- Specify a manipulation-resistant valuation method before implementing share issuance.
- Do not use a single instantaneous spot quote as the sole NAV source.
- Consider on-chain time-weighted observations collected across multiple slots, multiple independent liquidity venues where available, conservative bid-side executable liquidation value, route depth/price-impact limits, and explicit deviation checks.
- Define what happens when observations are missing, stale, newly created, inconsistent or too thin. Fail closed for share issuance rather than silently falling back to an untrusted quote.
- Define how the protocol values assets with no current executable USDC route. Do not let an unavailable route turn into a guessed price.
- Model the attack cost and expected loss for each enabled asset at the maximum intended vault size.

A TWAP is not a magic oracle: a thin or illiquid pool can still be manipulated over a sufficiently long window. The chosen observation window and liquidity thresholds need a threat-based justification.

**Tests required**
- Same-transaction spot manipulation cannot change the NAV used for share issuance.
- Single-pool manipulation and thin-liquidity cases are rejected or conservatively discounted.
- Stale observations, missing observations, route disagreement and price jumps fail closed.
- A newly enabled asset cannot be valued from an immature observation history.
- Fuzz and invariant tests cover rounding at extreme prices and balances.

### SEC-03 — Jupiter Router terminology and CPI trust boundary are not precise enough

**Severity: Critical integration blocker**

**Affected design:** “Jupiter Metis on-chain Router” and Swap API V2 `/build` as a route-construction aid.

Jupiter's current documentation describes Swap API V2's **Router path** as a way to obtain raw instructions using Metis on-chain routing, with CPI composability. The `/build` endpoint is an off-chain instruction builder; it is not itself the on-chain program that the vault invokes. The older Metis custom-swap guide is marked deprecated in favor of Swap API V2. The current docs distinguish Router (`/build`) from Meta-Aggregator (`/order` + `/execute`). See sources below.

**Risk**
The implementation team could mistakenly treat the API, routing engine and on-chain swap program as one stable interface. It could also pass too many accounts or an insufficiently constrained instruction to a CPI program that has been given PDA signing authority.

If a CPI target or downstream route can move vault-owned tokens to an unintended destination, the result can be direct loss of funds. The exact risk depends on the deployed program, instruction variant, account metas and route semantics, none of which are pinned in the current spec.

**Required changes**
- Identify the exact on-chain program ID that receives the CPI and the exact instruction discriminator/data layout.
- Pin the source/IDL revision, deployment and upgrade-authority status used for the target cluster.
- Document which `/build`-returned instruction is CPI-eligible. Do not blindly forward compute-budget, setup, cleanup, tip or arbitrary auxiliary instructions as nested CPIs.
- Define a strict parser/validator for the accepted swap instruction and all account metas.
- Validate input mint, output mint, source account, destination account, token program, signer PDA, amount, minimum output and all writable remaining accounts.
- Establish exactly which downstream programs the Jupiter program may invoke and how that route set is constrained.
- If the CPI target remains upgradeable, treat its upgrade authority as a material external trust dependency. Consider isolating each operation's exact input and output in operation-specific escrow accounts so a compromised integration cannot access the whole portfolio. This limits blast radius; it does not remove the trust dependency.
- If the required constraints cannot be verified, do not grant the Jupiter program authority over shared vault token accounts. Stop and choose a narrower independently audited adapter.

**Tests required**
- Wrong program ID, wrong instruction variant, wrong mint, wrong source/destination, wrong signer, extra writable account and unsupported downstream route are rejected.
- Route cannot direct output to a caller-controlled account.
- Only the exact amount reserved for the operation is exposed to the CPI.
- CPI failure or slippage failure rolls back the step and its status.
- Pinned deployed program behavior is tested on a local validator/fork.

### SEC-04 — Permissionless Recovery Ticket swaps can be griefed with poor execution

**Severity: High**

**Affected design:** Anyone may continue ticket swaps from ticket escrow.

**Attack scenario**
A third party submits a valid but economically poor route, with a very low or zero minimum output, consuming the ticket's remaining asset at an unacceptable price. The transaction is permissionless and technically valid, but the ticket owner suffers the loss.

**Required changes**
- Store an owner-approved minimum output or minimum execution-price rule for every ticket asset.
- Permissionless executors may choose a route, but cannot weaken the stored floor.
- If the owner wants to relax a floor, require an owner-signed update with a clear event and a monotonic version/nonce.
- Apply the same downstream program, pool, mint and destination restrictions as ordinary redemption.
- Do not treat a quote supplied by the executor as a price guarantee.

**Tests required**
- An executor cannot lower the ticket's minimum output.
- An executor cannot redirect proceeds or substitute a token.
- A route below the stored floor fails without consuming the ticket balance.
- Replaying a successful ticket step cannot execute it twice.

### SEC-05 — Aggregate minimum-USDC promise conflicts with the Recovery Ticket outcome

**Severity: High**

**Affected design:** Normal redemption checks aggregate minimum USDC after all assets are sold; ticket conversion moves accumulated proceeds and residual tokens to a ticket and burns shares.

**Problem**
Suppose all assets are sold, but total USDC is below the user's requested minimum. Normal finalization must fail. The design does not clearly say whether the owner may then convert to a ticket and claim the lower accumulated USDC. If that is allowed without explicit consent, the original minimum-output promise is misleading.

**Required changes**
- Define the aggregate minimum as a hard condition for normal completion.
- Define a separate explicit owner consent for accepting a ticket whose accumulated USDC may be below that minimum.
- Make clear that ticket conversion transfers the residual claim and burns the corresponding shares; it is not a guarantee of the original minimum.
- Consider storing both the original minimum and the owner-approved recovery terms in the ticket.
- Do not silently lower the minimum during retries or conversion.

**Tests required**
- Normal finalization below aggregate minimum always fails.
- A below-minimum ticket conversion cannot occur without the documented consent condition.
- The owner cannot accidentally sign a normal completion instruction that accepts a below-minimum result.
- Events and client-visible state distinguish “minimum met” from “owner accepted recovery claim”.

### SEC-06 — Deposit partial-execution recovery is a funds-locking blocker

**Severity: High**

**Affected design:** Staged deposit swaps; expiry policy remains unresolved.

**Attack/failure scenario**
A deposit buys two assets, then a third asset loses its route or liquidity. The operation cannot complete. If the vault retains the global lock, all other users may be blocked. If the program simply refunds the original USDC amount, the acquired assets are not accounted for and cannot be refunded as USDC without a route.

**Required changes**
- Design deposit recovery before coding the deposit state machine.
- Preserve a precise claim to all residual USDC and already-acquired tokens.
- Provide permissionless expiry/recovery triggering after a fixed deadline.
- Choose explicitly between deterministic unwind-to-USDC with a claim fallback and a deposit Recovery Ticket with owner-controlled USDC/in-kind claim. Never assume every asset can be sold back.
- Ensure shares are not minted until deposit finalization and the final contribution value is established.
- Ensure staged assets never enter active NAV before finalization.

**Tests required**
- Every possible partial-deposit step can be recovered without loss of claim.
- No one can claim the same staged token or USDC twice.
- Deposit expiry cannot leave the global lock held indefinitely.
- A route that becomes unavailable after preflight does not trap the deposit.

### SEC-07 — First-deposit and unsolicited-donation accounting are underspecified

**Severity: High**

**Affected design:** Initial share ratio is still an example; NAV is derived from vault holdings.

**Risk**
Vault share systems can suffer first-deposit/inflation and rounding attacks if the initial share price is attacker-controlled, if tiny deposits mint zero shares, or if unsolicited token transfers are mixed into NAV without a documented donation policy. The implementation plan currently mentions first-deposit ratio and rounding but does not define a complete defense.

**Required changes**
- Define a fixed initial share scale and minimum initial deposit.
- Use virtual shares/assets or another formally justified empty-vault protection.
- Reject deposits that would mint zero shares or less than a documented minimum.
- Define how unsolicited transfers to vault token accounts are treated. They must not be accidentally double-counted or used to manipulate the contribution/NAV calculation.
- Keep an internal accounting ledger for operation reservations and recognized balances, reconciled against actual token balances. Actual token balance deltas remain authoritative for each swap, but raw balances alone should not silently define every accounting category.
- Define rounding direction so users cannot systematically extract dust from the vault.

**Tests required**
- First-deposit dust and donation/inflation attack simulations.
- Direct unsolicited transfers before and after initialization.
- Tiny deposit, tiny redemption and zero-share/zero-output cases.
- Rounding-conservation properties over long sequences of deposits and redemptions.

### SEC-08 — Token issuer controls and Token-2022 extensions can defeat the vault's assumptions

**Severity: High**

**Affected design:** Token compatibility and asset onboarding.

**Risk**
A token may be frozen, paused, transfer-restricted, subject to a transfer hook, transfer fee, permanent delegate or other extension. A vault can be non-custodial at the protocol layer while an issuer or extension authority still freezes or seizes the underlying asset, blocks sales, or changes effective transfer amounts.

Token-2022 supports extensions including transfer fees, permanent delegate, transfer hooks, pausable/default account state and other features. These need an explicit compatibility policy rather than a generic “Token-2022 supported” flag.

**Required changes**
- For every configured mint, inventory mint authority, freeze authority and every enabled extension-specific authority.
- Reject unsupported extensions by default.
- Explicitly decide policy for transfer fees, transfer hooks, permanent delegates, pausable/default-frozen state, confidential transfers, CPI guard and any extension that changes balance/transfer semantics.
- Verify issuer, redemption restrictions, legal/blacklist controls and the ability to transfer from a PDA-controlled account.
- Revalidate mint configuration at onboarding and document which authorities can still change after deployment.

**Tests required**
- Each supported extension has explicit tests.
- Unsupported extension mints are rejected at initialization.
- Fee-on-transfer amounts use actual received balance deltas.
- A token that becomes frozen or paused causes a recoverable failure, not accounting corruption.

### SEC-09 — Queue snapshots and reservation accounting can become stale or double-counted

**Severity: High**

**Affected design:** Users may open/queue operations; NAV/supply snapshots; proportional reservations; one active operation.

**Risk**
If a request waits in a queue but snapshots NAV or share supply at opening, its valuation may be stale when it becomes active. If reserved assets are included in another redemption's available balance, or excluded from NAV while locked shares remain in supply, share value can be distorted. The current document does not fully define the queue activation boundary.

**Required changes**
- Take the authoritative NAV/share-supply snapshot when an operation becomes active, not when it is merely queued.
- Track requested, reserved, staged, live, and ticketed balances as separate accounting categories.
- Define whether reserved redemption assets remain in active NAV while locked shares remain in supply; keep numerator and denominator treatment consistent.
- Never allow two redemptions to reserve the same token units.
- Recompute or validate the share fraction against the active supply at activation.
- Keep operation sequence/nonce and explicit status transitions to prevent replay and stale finalization.

**Tests required**
- Queue wait does not preserve an obsolete NAV snapshot.
- Two redemptions cannot reserve the same assets.
- Pending deposits and tickets cannot be counted in active NAV.
- Locked shares and reserved assets remain consistently represented through every state transition.

### SEC-10 — Upgradeability is an explicit trust and recovery risk

**Severity: Medium–High**

**Affected design:** Vault program upgrade authority is expected to be revoked after audit.

**Risk**
Before upgrade authority is revoked, whoever controls it can deploy new code that changes asset movement, share minting or redemption behavior. After revocation, critical bugs may be permanently unpatchable.

**Required changes**
- Use a transparent, tightly controlled multisig/timelock for any pre-immutability upgrade authority.
- Publish and verify program builds and deployment artifacts.
- Define an emergency response and migration plan before mainnet.
- Revoke upgrade authority only after independent audit, deployment rehearsal and explicit owner acceptance of immutability trade-offs.
- Do not represent the system as trustless while a single operator can upgrade the vault program.

**Tests / release checks**
- Verify deployed program ID, ProgramData account and upgrade authority.
- Reproduce the build and compare the deployed artifact.
- Confirm the post-revocation program cannot be upgraded.

### SEC-11 — Fixed allocation on new deposits does not maintain portfolio target weights

**Severity: Medium (economic/design issue, not a direct exploit)**

**Affected design:** Fixed target allocation per deposit; no automatic rebalancing.

If the underlying assets appreciate or decline at different rates, the actual portfolio weights drift. Investing every new deposit according to the target percentages does not restore the whole portfolio to those percentages.

**Required changes**
- State clearly that the percentages are deposit-allocation weights, not guaranteed ongoing portfolio weights.
- Display actual weights and drift to users.
- Decide whether deposits should instead allocate toward underweight assets (this would be a different strategy and requires explicit approval).
- Do not describe the portfolio as continuously maintaining target weights if rebalancing is disabled.

## 3. Additional implementation controls

These controls should be added to the implementation checklist:

- Use canonical PDA derivation and validate every supplied PDA against the expected seeds and canonical bump.
- Validate account owner, mint, token program, authority, writable/signer privileges and expected token-account type for every token account.
- Reject unexpected duplicate accounts where duplication could change accounting or CPI semantics.
- Use checked arithmetic and explicit rounding on every amount conversion.
- Use a unique operation ID/nonce and monotonic step sequence for each deposit, redemption and ticket.
- Change operation status in a transactionally safe order; a failed CPI must leave both token balances and operation state unchanged.
- Keep source and destination token accounts operation-scoped wherever possible, so a route CPI only receives the funds required for that step.
- Define rent payer, account close authority and dust disposition for every operation and ticket account.
- Emit events for all state transitions, but never rely on off-chain events as the authoritative accounting record.
- Test worst-case account count, compute units, CPI depth and transaction size for the complete portfolio basket.

## 4. Recommended revised execution order

Do not jump straight from the current implementation plan into DEX CPI code. Use this order:

1. Resolve SEC-01 and SEC-06: bounded operations, permissionless expiry and deposit/redemption recovery.
2. Resolve SEC-02 and SEC-07: NAV valuation, first-deposit protection, donation policy and rounding.
3. Resolve SEC-04 and SEC-05: ticket execution floors and minimum-USDC consent semantics.
4. Resolve SEC-03: verify the exact Jupiter Router/CPI target and define the strict instruction/account validator.
5. Resolve SEC-08: choose exact mint/extension policy and issuer-risk acceptance for every asset.
6. Resolve SEC-09 and SEC-10: queue activation, reservation ledger and upgrade authority process.
7. Convert every accepted rule into unit, property, state-machine and adversarial tests.
8. Only then begin implementation.

## 5. Sources consulted

- Jupiter Swap API V2 overview: https://developers.jup.ag/docs/swap
  - Distinguishes Meta-Aggregator from Router; Router uses Metis on-chain routing and returns raw instructions for custom transaction composition/CPI. The old Metis custom-swap guide is deprecated.
- Jupiter Swap API V2 build documentation: https://developers.jup.ag/docs/swap/build
  - Describes the off-chain `/build` endpoint and raw instruction construction. The API response is not itself a trusted on-chain valuation or authorization source.
- Solana CPI documentation: https://solana.com/docs/core/cpi
  - Explains PDA signing, CPI account privileges, shared compute budget and CPI limits.
- Solana PDA derivation: https://solana.com/docs/core/pda/pda-derivation
  - Recommends canonical bump validation to avoid alternate PDA addresses.
- Solana security guidance: https://solana.com/docs/tools/security
  - Recommends allowlisting exact token and program addresses and validating account types.
- Token-2022 extension reference: https://www.solana-program.com/docs/token-2022
  - Lists extensions including transfer fees, permanent delegate, transfer hooks, pausable and default account state.
- Token-2022 extension guide: https://www.solana-program.com/docs/token-2022/extensions
  - Explains extension-specific transfer behavior and authority implications.

## 6. Scope and limitations

This is a document-based architecture review. It did not inspect or execute an Anchor program, because the repository currently contains design documentation rather than an implemented program. It did not verify the current Jupiter program deployment, upgrade authority, IDL, or route CPI semantics on-chain. Those are explicit verification gates, not assumed facts.

**Verdict:** The overall architecture is a reasonable starting point, but the current implementation plan is not yet security-complete. Resolve the Critical and High findings before coding the corresponding state machines or enabling real assets. An independent Solana security audit remains necessary before mainnet deployment.


## 7. Follow-up remediation status

A separate remediation pass has translated the findings into concrete architecture rules and updated the design specification and implementation plan:

- Remediation decisions: docs/superpowers/reviews/2026-11-11-solana-portfolio-vault-security-remediation.md
- Updated design: docs/superpowers/specs/2026-11-11-solana-portfolio-vault-design.md
- Updated implementation plan: docs/superpowers/plans/2026-11-11-solana-portfolio-vault-implementation-plan.md

The principal design changes are: remove the long-lived global operation lock; isolate every operation in its own escrow; impose an immutable seven-day operation deadline; add permissionless Deposit Recovery Tickets; let anyone trigger recovery while preserving the original user's ownership; enforce owner-set per-asset execution floors; require explicit consent for below-minimum USDC claims; use fail-closed multi-venue on-chain TWAP valuation; add virtual share/accounting offsets and a recognized-balance ledger; adopt a strict v1 token admission policy; and isolate all DEX CPI funds in operation-specific accounts.

**Status:** The documentation-level remediations are applied. SEC-02, SEC-03 and SEC-08 remain asset/integration verification gates, and every design mitigation still requires implementation and adversarial tests. This follow-up does not upgrade the original review into a source-code audit.
