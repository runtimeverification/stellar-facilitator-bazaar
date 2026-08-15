# Technical Architecture: A Safe and Extensible x402 Facilitator with Bazaar Discovery for Stellar

**Runtime Verification · SCF #45 Build Award · RFP Track**

> This document describes the design we intend to build and techniques used to achieve the stated deliverables in the RFP proposal. Details may change as implementation proceeds. 
> Alongside the architectural proposal, we'd like to highlight the investigation and documentation of the security properties stated in this document. This will be used to guide the development of the facilitator and its components, and can also be a reference document to other developers working on similar systems or auditors reviewing them. 

---

## 0. Document outline

| § | Section |
|---|---|
| 1 | **Component architecture** |
| 2 | **Settlement core and the adapter seam** |
| 3 | **Protocol adapters: x402 and MPP** |
| 4 | **The `upto` scheme and its Soroban contract** |
| 5 | **Bazaar: cataloging and catalog integrity** |
| 6 | **Discovery search and its evaluation harness** |
| 7 | **MCP discovery interface** |
| 8 | **Threat model and security properties** |
| 9 | **Operations: configuration, monitoring, self-hosting** |

---

## 1. Component architecture

We propose a modular architecture segregated by responsibilities and trust boundaries, so that parts of it can be tested both in isolation and in combination. The approach also makes the architecture flexible and extensible.

### 1.1 Structure

The system layers by criticality. Reading downward is increasing consequence of failure: a search outage is an inconvenience, a settlement defect is a loss.

```
   buyer / AI agent                                  seller's service
          │                                                 │
     ─────┼──────────── TB1 ────────────      ──── TB2 ─────┼─────
          │                                                 │
  ┌───────▼─────────┐                        ┌──────────────▼──────┐
  │   MCP server    │                        │  seller SDK         │
  │  holds no keys  │                        │  declare · lint ·   │
  │  (M1)           │                        │  validate unpaid    │
  └───────┬─────────┘                        └──────────────┬──────┘
          │                                    discovery    │
          │                                    metadata     │
  ┌───────▼──────────────────────────────────────────┐      │
  │             PROTOCOL ADAPTERS                    │      │
  │   x402 (exact · upto)      MPP (Charge)          │◀─────┘
  └───────┬──────────────────────────────────────────┘
          │   PaymentIntent · AuthorizationBundle · SettlementOutcome
  ┌───────▼──────────────────────────────────────────┐
  │             SETTLEMENT CORE                      │   ← criticality:
  │   invariant enforcement · non-custody · replay   │     unconditional
  └──┬──────────────┬───────────────┬────────────────┘
     │              │               │
 ┌───▼──────┐  ┌────▼────────┐  ┌──▼───────────────┐
 │  asset   │  │  channel    │  │ Soroban client   │        TB4
 │  policy  │  │  leases     │  │ RPC failover ·   │  ───────────────▶
 │          │  │  (fenced)   │  │ simulation ·     │      Stellar
 │          │  │             │  │ hash recovery    │      network
 └──────────┘  └─────────────┘  └──┬───────────────┘
                                   │ settled outcome (observed truth)
                       ┌───────────▼─────────────┐
                       │     CATALOG WRITER      │   ← subordinate to
                       │  owner = settled        │     settlement (C1)
                       │  recipient (C2, C3)     │
                       └───────────┬─────────────┘
                                   │
              ┌────────────────────▼──────────────────┐
              │  DISCOVERY API · SEARCH INDEX         │   ← subordinate
              │  + evaluation harness                 │     to both
              └────────────────────┬──────────────────┘
                                   │
                     ─────── TB3 ──┼──────────▶  agents consume results
                                   │
              ┌────────────────────▼──────────────────┐
              │  OBSERVABILITY  metrics · alerts ·    │
              │  status · TTL and sponsor monitoring  │
              └───────────────────────────────────────┘
```

### 1.2 Responsibilities

| Component | Owns | Explicitly does not own |
|---|---|---|
| **Protocol adapters** | Wire framing: parse requests into a `PaymentIntent`, encode challenges, extract authorization credentials, encode outcomes, map errors into protocol vocabulary | Any settlement invariant. Adapters cannot weaken non-custody, replay protection, binding, or fee ceilings (§2.3) |
| **Settlement core** | Authorization-entry validation, simulation discipline, invariant enforcement, replay state, submission idempotency | Protocol framing; on-chain settlement shape |
| **Asset policy** | Allowlist per network, decimals read from the token contract, trustline validation at verify *and* settle | Which assets an operator should choose, which is configuration |
| **Channel leases** | Fenced sequence-number supply, pool sizing from measured cost | Fee payment (decoupled via fee-bump signer) |
| **Soroban client** | RPC access with provider failover, record and enforcing simulation, submission, hash-based recovery | Deciding validity, since we treat an RPC as potentially degraded or lying (T25) |
| **Catalog writer** | Settlement-triggered cataloging, schema validation with soft drop, ownership binding, `EXTENSION-RESPONSES` reporting | Any ability to affect the payment it observes (C1) |
| **Discovery API + search** | Filtered listing, deterministic ordering, cursor pagination, retrieval and ranking, evaluation harness | Constraint decisions, which are deterministic and upstream of ranking (S1) |
| **MCP server** | Tool surface, orchestration of the discover → pay → retry loop, error classification | Keys, funds, and signing (M1) |
| **Seller SDK** | Metadata declaration, startup lint, validate-without-paying | Trust. Everything it emits is untrusted input downstream |
| **Observability** | Settlement, sponsor-balance, channel-pool, TTL, and discovery signals with alert thresholds | Thresholds themselves before measurement exists (§9.4) |

### 1.3 Trust boundaries

Four boundaries carry adversarial traffic. Each is named in the threat model with its controls and tests.

| | Boundary | Adversary | Primary concern |
|---|---|---|---|
| **TB1** | Buyer / agent → facilitator | Hostile buyer; hostile custom account | Fee drain, replay, verification bypass (T1–T10) |
| **TB2** | Seller metadata → catalog | Hostile seller | Listing and price spoofing, traversal, flooding (T11–T17) |
| **TB3** | Catalog / search → agent | Hostile listing *content* | Prompt injection into agent context, induced overspend, result manipulation (T18–T22) |
| **TB4** | Facilitator → Stellar RPC | Degraded or lying infrastructure | Lost responses, double submission, false simulation results (T5, T25) |

**TB3 is the boundary we consider most under-examined in this field.** TB1 and TB2 are conventional service-hardening problems with well-understood controls. TB3 differs in kind. The adversary aims at *our users through us*, using content we faithfully store and serve. A facilitator that is perfectly secure in the conventional sense can still be an effective delivery mechanism for an attack on the agents that trust it. That is why untrusted seller data carries its provenance structurally, why seller text never reaches an instruction position (M3), and why we staff AI-security specialists on the project instead of consulting them at the end.

## 2. Settlement core and the adapter seam

### 2.1 Two orthogonal axes

Payment behaviour varies along two independent axes:

- **Protocol**: how a payment is framed over HTTP. Header names, challenge encoding, credential shape, error vocabulary, and whether the client or the server broadcasts. x402 and MPP differ here.
- **Settlement shape**: what happens on-chain. A fixed-amount SAC `transfer`, or a variable-amount settlement bounded by a signed ceiling and mediated by a contract. `exact` and `upto` differ here.

MPP `Charge` in Pull mode is a different protocol performing the same settlement shape as x402 `exact`. The `upto` scheme is the same protocol performing a different settlement shape. We therefore factor protocol adapters onto one axis and settlement strategies onto the other, over a Stellar mechanics core that owns neither.

```
                 ┌──────────────── protocol axis ────────────────┐
                 │  x402 adapter          MPP Charge adapter     │
                 │  (challenge/402,       (Pull + Push credential│
                 │   PaymentPayload)       modes)                │
                 └───────────────────────────────────────────────┘
                                     │
                        canonical PaymentIntent /
                        AuthorizationBundle / SettlementOutcome
                                     │
                 ┌──────────── settlement axis ──────────────────┐
                 │  ExactSettlement        UptoSettlement        │
                 │  (SAC transfer)         (bounded, contract)   │
                 └───────────────────────────────────────────────┘
                                     │
                 ┌───────────────────────────────────────────────┐
                 │        Stellar settlement core                │
                 │  auth-entry validation · simulation discipline │
                 │  asset & trustline policy · fee sponsorship    │
                 │  sequence/channel management · submission &    │
                 │  idempotency · ledger-bound expiry             │
                 └───────────────────────────────────────────────┘
```

Adding MPP touches only the protocol axis. Adding `upto` touches only the settlement axis.

### 2.2 What the core owns

**Authorization-entry validation.** The client signs Soroban authorization entries, never a pre-signed transaction, and the facilitator rebuilds the transaction around them. Validation is structural and exact:

- exactly one `invokeHostFunction` operation, invoking the expected contract and function with the expected arguments;
- an authorization tree matching the expected shape, with no extra or unexpected sub-invocations;
- the facilitator's own accounts nowhere as the value source, neither as `from` nor as a signer of the payment authorization;
- signed material binding asset, amount or ceiling, recipient, and resource;
- replay state checked before any fee is spent.

**Simulation discipline.** Three phases, and the ordering is a security property:

1. **Record simulation** derives the footprint and authorization tree.
2. **The client signs** the resulting authorization entries.
3. **Enforcing simulation** executes custom-account `__check_auth`, then we rebuild and submit.

Signature verification is explicit and does not rely on simulation success. A recording simulation that passes proves nothing about authorization.

**Asset and trustline policy.** Decimals come from the token contract, never assumed to be 7. We validate trustline state at `verify` and re-validate at `settle`, since it can change in between, with distinct machine-readable reasons for missing, unauthorized, `AUTHORIZED_TO_MAINTAIN_LIABILITIES`, paused, clawback-enabled, and insufficient-balance conditions. Testnet accepts any SEP-41 asset. Mainnet uses a curated, operator-configurable allowlist, since sponsored fees make a pathological token contract a fee-drain vector (§8).

**Fee sponsorship.** The facilitator sponsors network fees, so the buyer holds only the payment asset and needs no XLM. A fee-bump signer decouples fee payment from sequence-number management, so concurrent settlements do not contend on one account's sequence.

**Sequence and channel management.** Stellar admits roughly one transaction per account per ledger, so agent bursts contend. A leased channel-account pool supplies sequence numbers, with leases fenced so a slow or retried submission cannot reuse a sequence another submission has taken.

**Submission and idempotency.** We persist the final envelope and its hash **before** submission. Recovery polls by hash and never blindly resubmits, so a lost RPC response cannot become a double settlement.

**Ledger-bound expiry.** `signatureExpirationLedger` derives from the operator's configured `maxTimeoutSeconds` (≈12 ledgers per 60 seconds). Timeout, fee ceiling, and expiry are configured as one coupled control: a long expiry inflates nonce rent, a short one causes spurious rejections under load.

### 2.3 The seam contract

An adapter owns protocol framing and nothing else. It implements, in both directions:

| Direction | Responsibility |
|---|---|
| Inbound | Parse a protocol request into a canonical **PaymentIntent** (network, asset, amount or ceiling, `payTo`, resource binding, expiry) |
| Outbound | Encode the payment challenge (the HTTP 402 response) in the protocol's wire format |
| Inbound | Extract the **AuthorizationBundle** (signed auth entries, or a broadcast-transaction credential) from the protocol's payload |
| Outbound | Encode the **SettlementOutcome**, including whatever receipt or confirmation artifact the protocol requires |
| Both | Map internal rejection reasons onto the protocol's error vocabulary, with a non-null reason on every path |

Two invariants hold at the seam. **No adapter may weaken a settlement invariant**: non-custody, replay protection, binding checks, and fee ceilings live in the core and are not adapter-configurable. **Every internal rejection reason maps to a non-null protocol reason**, enforced at the seam so no adapter can silently drop one.

**Submission paths per settlement shape.** `exact` invokes the token's Stellar Asset Contract directly with an amount the payer signed. `upto` invokes our settlement contract, and the actual amount arrives at settle time, deliberately absent from the signed arguments. We start with a separate submission path for each shape while the `upto` contract's details settle (§4.2).

The separation covers transaction construction only. Non-custody checks, replay protection, binding, fee ceilings, submission idempotency, and hash-based recovery stay in the shared core with exactly one implementation each. During development we will evaluate merging the two paths, and will merge them if that costs no significant special-casing.

### 2.4 What this buys

| Change | Work required |
|---|---|
| x402 `exact` | Adapter + `ExactSettlement` (T#1) |
| x402 `upto` | New settlement strategy + contract only, adapter unchanged (T#2) |
| MPP `Charge` Pull | New adapter only, settlement unchanged (T#3) |
| MPP `Charge` Push | Same adapter, plus a verify-a-broadcast-transaction credential path (T#3) |
| Future scheme or protocol | One axis, not both |

---

## 3. Protocol adapters: x402 and MPP

Adapters own wire framing and nothing else; the seam contract they implement is defined in §2.3. This section covers what each adapter is responsible for, and what the second one costs. That cost is the empirical test of the two-axis claim in §2.1.

### 3.1 x402 adapter

**Surface.** `POST /verify`, `POST /settle`, `GET /supported`, on `stellar:testnet` and `stellar:pubnet` addressed by CAIP-2 identifiers.

**Conformance is the design constraint, not a later check.** The RFP makes wire-level conformance against *unmodified* canonical clients a hard acceptance criterion, which fixes several things that might otherwise look like implementation latitude:

- the canonical `payload: { transaction }` is accepted **verbatim**, not in a convenient variant;
- `/supported` returns the correct Stellar `extra` contract, advertising `areFeesSponsored` when the operator sponsors fees;
- every rejection carries a non-null, machine-readable reason (§7.3), because a canonical client that receives a null reason has no path forward;
- we advertise both schemes per network, so a client can discover `upto` availability instead of probing for it.

`/supported` is therefore a conformance surface, not a convenience endpoint. It is the machine-readable statement of what this deployment accepts, and the RFP checks it directly.

**Scheme dispatch is where the axes cross.** The adapter maps `(scheme, network)` onto a settlement strategy, sending `exact` to the direct SAC transfer and `upto` to the bounded-draw contract path of §5. It does nothing else differently between them. The protocol framing is identical for both; only the settlement shape changes.

### 3.2 MPP adapter for the `Charge` intent

We build on the official **`@stellar/mpp`** SDK and do not implement the specification ourselves, the same way we build on `@x402/stellar` for x402.

MPP's `Charge` intent settles each request individually through a Soroban SAC `transfer`, It offers two credential modes that differ in *who broadcasts*, and architecturally that difference matters more than the framing does.

**Pull mode (default).** The client prepares and signs Soroban authorization entries; the server broadcasts. Optionally the server rebuilds the transaction so the client needs no XLM.

This is, mechanically, what x402 on Stellar already does. Same signing model (authorization entries, not pre-signed transactions), same rebuild-and-submit, same sponsorship. The adapter is therefore close to pure framing translation over an unchanged settlement core, which is precisely the outcome §2.1 predicts and the reason MPP is affordable at all.

**Push mode.** The client broadcasts, then presents a `signedHash` credential: the transaction hash plus a signature over it.

Push inverts our posture from *authorizing a future action* to *observing a settled fact*, so it needs its own validation path:

1. the transaction exists on-chain and succeeded;
2. it actually performs the payment the intent describes, meaning asset, amount, and recipient, which requires inspecting the transaction and never trusting the hash alone;
3. the signature binds the credential to the payer account;
4. the transaction has not already been redeemed against another request;
5. it falls inside the validity window.

The security profile is a mirror image of Pull, and worth stating because it is easy to get backwards:

| | Pull | Push |
|---|---|---|
| Who pays the network fee | Facilitator (sponsored) | Client |
| Fee-drain exposure | Present, and the main surface (T2, T3, T4) | **None**, since we never submit |
| Replay / reuse exposure | Bounded by the host nonce and single-use authorization | **New surface**. The same settled transaction could be presented to several sellers, so consumed hashes must be tracked and bound to the intent |

Push removes our largest attack surface and introduces one Pull does not have. Neither mode is strictly safer; they trade one class of risk for another, and both need their own controls.

### 3.3 What the second protocol actually costs

The concrete payoff of §2.1, stated as work touched:

| Combination | Adapter | Settlement strategy | Stellar core |
|---|---|---|---|
| x402 `exact` | new (T#1) | new (T#1) | new (T#1) |
| x402 `upto` | **unchanged** | new contract + strategy (T#2) | **unchanged** |
| MPP `Charge` Pull | new, framing only (T#3) | **unchanged** | **unchanged** |
| MPP `Charge` Push | same adapter + credential path (T#3) | observation path | **unchanged** |

Two protocols and two settlement shapes, with the Stellar mechanics core written once. We exercise each axis independently and at a different point in the schedule, so we demonstrate the extensibility claim twice instead of asserting it once.

### 3.4 MPP `Session`

MPP's `Session` intent uses unidirectional payment channels. The client deposits once, signs cumulative off-chain commitments, and the server settles by closing the channel.

We are not implementing it here. It falls outside this RFP's scope, and supporting it would mean adding modules this architecture does not otherwise need: channel lifecycle, cumulative commitment accounting, dispute handling, and persistent channel state held to the same integrity standard as the settlement path. It would also introduce a deadline whose breach costs value, since an open channel must be closed in time.

Session is well-suited to a follow-on engagement, where it can be given the design attention its complexity deserves, and considering the modular design proposed here, it can be slotted in without touching the settlement core or the x402 adapter.

### 3.5 MPP resources in the catalog

If MPP-protected resources are catalogued alongside x402 ones, discovery metadata needs a **protocol dimension** and search results must state which protocol a resource speaks. Otherwise an agent can discover a resource it has no way to pay. This field is cheap to design in at T#1 and awkward to retrofit at T#3, so the schema carries it from the start and it is populated when the MPP adapter lands. Whether it is *exposed* as a filter on `/discovery/resources` before MPP ships is a smaller, later call.

### 3.6 Conformance, stated honestly per protocol

- **x402** is conformance-tested against unmodified canonical clients and the official e2e suite, on both networks, with settled transaction hashes published per scheme per network. This is the RFP's hard criterion and we treat it as pass-or-fail.
- **MPP** has a younger ecosystem: an official SDK and a testnet reference deployment exist, but not the same body of independent canonical clients that makes x402 conformance a strong oracle. We therefore conform against `@stellar/mpp` as the reference implementation plus the published specification, and we say so plainly instead of implying parity with the x402 conformance story. Overstating this would be the easiest and least defensible claim in the proposal.

## 4. The `upto` scheme and its Soroban contract

The `upto` scheme lets a payer authorize a **ceiling** and settle for **actual usage** below it, with one signature. This is the missing primitive for metered pricing, covering per-token inference, per-row data delivery, and per-second compute, where the price is not known when the payment is authorized.

### 4.1 Why a contract is required

A SEP-41 allowance alone cannot express the scheme:

- **No recipient binding.** An allowance authorizes a *spender* to move the payer's funds; it says nothing about *who receives* them. A facilitator holding an allowance could direct funds anywhere.
- **No terminal single-settlement.** An allowance persists until exhausted or expired and admits repeated partial draws. "Settle exactly once, for the actual amount" is not expressible.
- **No bound on the settled amount relative to the authorized intent.** Nothing ties a draw to the specific resource and terms the payer agreed to.

A minimal contract supplies exactly these three missing guarantees and nothing else.

### 4.2 Contract shape: minimal by design

- **One entry point.** A single `settle_upto` function.
- **No admin, no upgrade path, no pause, no allowlist, no persistent state.** Immutable once deployed.
- **Never holds a token balance** (see §4.4).
- **Replacement, not upgrade.** Because the contract is immutable and administrator-free, anyone may deploy a replacement, and only the facilitator operator activates a new address through configuration. Governance is therefore not an attack surface.
- **No settlement hook.** A hook would add a hostile-callee surface, since it can burn CPU or fail mid-settlement, and it would widen the property surface the test suite has to cover.

Minimality is a security decision. It shrinks the property surface enough for the Komet suite in §4.5 to cover it meaningfully, removes upgrade and admin keys as targets, and lets the deployed Wasm hash be checked against a clean rebuild.

**Single use without contract storage.** `require_auth_for_args` consumes a Soroban host nonce, and we rely on that nonce for single settlement (I5). The contract therefore keeps no settlement-identifier storage, which avoids the rent and maintenance overhead that persistent state carries. If testnet measurement shows a storage-based approach is materially more efficient, we will adopt it instead.

### 4.3 Binding a ceiling without binding the amount

The crux of the scheme: the payer must authorize the ceiling and the terms, but cannot sign the actual amount, which does not exist yet at authorization time.

The payer's signature covers the bound arguments via `require_auth_for_args`, **excluding `actual_amount`**: the token, the recipient `payTo`, `max_amount`, the deadline, and a settlement identifier. The facilitator supplies `actual_amount` unsigned at settle time, and the contract asserts it against the signed ceiling. The payer's authorization tree carries the SEP-41 `approve` as a nested sub-invocation, so a single signature covers both the allowance and the settle call.

### 4.4 Design fork: no custody window

There are two ways to realise settlement, and the choice is consequential.

| | **(a) Pull-and-refund** | **(b) Bounded draw**, *our design* |
|---|---|---|
| Mechanism | Pull `max_amount` into the contract, pay `actual` to `payTo`, refund `max − actual` to the payer, atomically | `approve(contract, max_amount)`, then draw exactly `actual` from payer to `payTo`; the unused remainder is never moved and its allowance lapses at `live_until_ledger` |
| Token transfers | Three | One |
| Contract ever holds funds | Yes, transiently | **No** |

We adopt **(b)**, for three reasons:

1. **Zero custody becomes structural.** The contract never appears as a token holder at any point in any invocation, so "never takes custody" is a property of the design, not a claim requiring proof of atomicity.
2. **No clawback exposure at the contract.** With clawback-enabled assets, design (a) creates a moment where clawable balance sits at a contract address. Design (b) has no such moment.
3. **One fewer failure mode.** Removing the refund leg removes an atomicity hazard. There is no partially-refunded state to reason about, because the remainder is never taken.

The cost of (b) is a residual allowance that persists until its expiry ledger. That is bounded, and bounding it correctly is precisely what the clock ordering in §4.6 enforces. Single-use (I5) prevents the residual allowance from being drawn again.

### 4.5 Invariants and the Komet property suite

These invariants are the specification, the test targets, and the content we contribute upstream. Komet, the property-based testing and fuzzing tool we built for Soroban, exercises each one with a property in place of a hand-written example.

| # | Invariant |
|---|---|
| I1 | **Value conservation.** The payer's balance decreases by exactly `actual`; `payTo` increases by exactly `actual`. |
| I2 | **Zero custody.** The contract's balance in any asset is zero before and after every invocation, under every call sequence. |
| I3 | **Bounds.** `0 ≤ actual ≤ max`. Any `actual > max` rejects with no state change and no transfer. |
| I4 | **Zero settlement still terminates.** `actual == 0` moves no value but consumes the authorization, so it cannot be replayed. |
| I5 | **Single use.** A given signed authorization settles at most once, including under interleaved or concurrent submission. |
| I6 | **Recipient binding.** Settlement to any address other than the signed `payTo` rejects. |
| I7 | **Asset binding.** Settlement in any token other than the signed one rejects. |
| I8 | **Temporal validity.** Settlement at or past the deadline rejects; the clock ordering in §4.6 holds. |
| I9 | **Atomicity.** If the draw fails for any reason, the invocation leaves no partial state. |
| I10 | **Auth-tree exactness.** An authorization tree containing extra or unexpected sub-invocations rejects. |
| I11 | **Numeric safety.** `i128` boundaries and negative amounts reject cleanly, with no wrapping and no panic. |

**Adversarial Soroban doubles**, themselves contracts, and the reason this work is Komet-shaped instead of unit-test-shaped:

- **SEP-41 mock tokens** exhibiting real hostile behaviour: `authorization_required`, paused, clawback-enabled, error-returning, and non-7-decimal.
- **Mock custom accounts** with a panicking `__check_auth` and a CPU-burning `__check_auth`, the latter to probe the fee-drain surface created by sponsorship (§8).

This is the strongest single technical claim available to us on the on-chain surface: the field's best current evidence for this contract is a few dozen hand-written unit tests, and no competitor property-tests or fuzzes it at all.

### 4.6 The three clocks

Three expiry times must be ordered, and getting it wrong fails either safely-but-annoyingly or unsafely:

```
allowance live_until_ledger  ≥  contract deadline  ≥  actual settlement ledger
```

All three derive from the operator's configured `maxTimeoutSeconds`. A window that is too long inflates rent on nonce and allowance state; too short causes spurious rejections under load. The boundaries are tested explicitly at before, equal to, and after the deadline.

### 4.7 Upstream contribution

We write the Stellar network specification for the `upto` scheme in x402 specification format and open it upstream, coordinating through the x402 Technical Steering Committee. The contribution covers the authorization-tree shape, the clock-ordering rule, the invariant list from §4.5 stated normatively, and test vectors, so any Stellar facilitator can implement the scheme and check itself against the same properties.

## 5. Bazaar: cataloging and catalog integrity

### 5.1 Cataloging as a consequence of settlement

The Bazaar has **no registration endpoint**. A resource enters the catalog because a payment for it settled while carrying the discovery extension. Two properties follow:

- **Spam resistance is economic.** A listing costs a real settled payment, so there is no free write path to defend.
- **Ownership is observed, not claimed.** We have just executed the settlement, so recipient, asset, network, and amount come from the transaction. The client-supplied `payTo` in discovery metadata is never trusted, and a seller cannot list a resource they do not own.

### 5.2 Catalog invariants

These are the specification and the test targets. §8 maps threats onto them.

| # | Invariant |
|---|---|
| C1 | **Cataloging never blocks or alters settlement.** A payment settles or fails on its own merits; no discovery-metadata problem can change that outcome. |
| C2 | **A listing's owner is the settled recipient**, derived from the transaction, never from client-supplied metadata. |
| C3 | **Advertised terms match settled terms.** Price, asset, and network on a listing are derived from the settlement, so a seller cannot advertise one price and charge another. |
| C4 | **No seller-supplied value is dereferenced server-side.** URLs, schema references, and other seller inputs are stored and served, never fetched or resolved by us. |
| C5 | **Pagination is stable.** A cursor walk under concurrent catalog writes yields neither duplicates nor gaps. |

### 5.3 Resource identity: HTTP and MCP are not the same shape

Both are first-class resource types, and they key differently:

- **HTTP resources** are identified by normalized URL together with the route template and method.
- **MCP tools** cannot be identified by URL alone, because a single MCP endpoint multiplexes many tools. Identity is the pair `(resource url, tool name)`.

Where two owners present the same identity, we **quarantine the entry for review**. Last-write-wins would be a listing-takeover primitive.

### 5.4 Validation: cataloging is subordinate to payment

C1 forces the ordering. We validate discovery metadata against its schema, and failures degrade the listing, never the payment:

- **Per-field soft drop** for optional metadata: keep what is valid, discard what is not.
- **Whole-listing drop** when identity or terms-critical fields are invalid, since a partially-populated listing on those fields would mislead, not merely under-inform.
- **Every outcome is reported** through the `EXTENSION-RESPONSES` header, success and failure alike, each with a specific, machine-readable, non-null reason. A seller must be able to learn *why* a listing was dropped without guessing.

### 5.5 Input-integrity controls

| Surface | Control |
|---|---|
| `routeTemplate` path traversal | Percent-decode **before** traversal checks, with bounded repeat-decoding to catch multi-layer encoding such as `%252e%252e`. Checking before decoding is the classic mistake here; the decode loop is bounded to avoid a decode bomb. |
| Seller-supplied URLs (icons, docs, endpoints) | **Never dereferenced server-side** (C4). We store and serve them, and the client decides whether to fetch. This removes SSRF as a category instead of filtering for it. |
| Schema references in metadata | Restricted to same-document fragments, so no external resolution is possible. |
| Catalog flooding | Per-recipient insert and update rate limits, plus bounded metadata size. Economic cost already raises the floor; these bound the ceiling. |
| Terms drift | Price, asset, and network taken from the settlement (C3), not from metadata. |

### 5.6 The agent-facing boundary: untrusted provenance and prompt injection

This is the surface we consider most under-examined in the field, and the reason RV's AI-security specialists are staffed on the project.

Seller-supplied text, meaning titles, descriptions, and tool documentation, flows through the catalog into **search results and MCP tool responses that autonomous agents consume**. Text arriving from an untrusted party and landing in an agent's context is an injection vector: a listing can attempt to instruct the agent reading it.

Our design position:

- **Provenance is explicit.** Seller-authored fields are carried and served as untrusted data, distinguishable from facilitator-derived fields (which come from settlement and are trustworthy). Consumers can tell the difference structurally, with no reliance on convention.
- **Structural separation, not sanitisation.** Seller text is never interpolated into instruction positions in MCP responses. We do not attempt to "clean" natural-language text of injection attempts, because that is not a solvable filtering problem; we constrain where untrusted text can appear instead.
- **Facilitator-derived fields drive ranking and filtering** wherever possible, so seller prose has limited influence over what an agent is shown.
- **Reviewed as an engagement, with findings that land somewhere.** The boundary gets a time-boxed AI-security assessment. A finding that reduces to a deterministic check becomes a test, and a finding that does not becomes documented design guidance.

It is also why we leave a neural reranker out of this scope: it would place a model that ingests seller prose directly in the agent-facing path, expanding precisely this surface.

### 5.7 `GET /discovery/resources`

Spec-conformant filters (`type`, `payTo`, `network`, `extensions`, `limit`, `offset`), combinable, with **deterministic total ordering** so that pagination satisfies C5 while the catalog is being written concurrently.

### 5.8 Off-chain index

The index is off-chain, as the RFP directs. The RFP separately requires that submissions demonstrate understanding of TTL and rent strategy for an *optional* on-chain registry, so we document the analysis instead of building the registry:

- **Two TTLs to maintain**, not one: the contract instance and the Wasm code entry, each needing rent extension before archival, and each an availability risk if missed.
- **Rent liability grows with catalog size.** Persistent per-entry storage means an accumulating obligation on a catalog designed to grow without registration friction.
- **There is no natural payer.** Charging sellers rent at listing time reintroduces the registration step that automatic cataloging exists to remove; absorbing it as operator makes liability unbounded in the catalog's own success.
- **Soroban resource limits bound reads and writes per invocation**, so on-chain filtering and pagination are expensive precisely where discovery needs them cheap.
- **Archival recovery is a transaction**, so a cold entry cannot be read until someone pays to restore it.

Conclusion: an on-chain registry adds a permanent, growing cost and two liveness obligations in exchange for guarantees the catalog does not need at this stage, since integrity already derives from settlement (C2, C3) and not from consensus over the index. What would change the calculus: listings becoming valuable enough that sellers willingly fund their own rent, or a federation requirement that needs a neutral on-chain root.

### 5.9 Seller-side helpers

Required by the RFP, and shaped by one observation: a seller should not have to **pay** to discover that their metadata is invalid.

- A declaration helper for emitting well-formed discovery metadata.
- A **startup lint** that warns on missing or weak metadata when the seller's service boots.
- A **validate command** that checks a listing would be accepted, including the `routeTemplate` and schema checks, against a live endpoint **without making a payment**.

### 5.10 Spec drift and interoperability

The Bazaar extension is still evolving, and the RFP requires both conformance tracking and interoperability with the wider ecosystem. Concretely: we pin the conformance suite to an explicit specification version, we track specification changes as issues with a documented triage path, and we test interoperability by running unmodified canonical clients against our deployment. Wire-level conformance to the published specification is what makes a resource we catalog readable by any other x402 implementation.

## 6. Discovery search and its evaluation harness

### 6.1 Correctness and relevance are different problems

An agent's query mixes two kinds of requirement, and treating them as one is the central design error available here.

- **Hard constraints**: network, asset, resource type, price bound, required extensions. A result that violates one of these is not *less relevant*; it is **wrong**. An agent that acts on it wastes a paid call or fails outright.
- **Semantic relevance**: "find me a weather API". Being approximately right is acceptable and unavoidable.

So the pipeline separates them: **constraints are enforced deterministically, relevance is ranked heuristically.** Ranking never gets the opportunity to trade a constraint away for a better-looking score. This gives us one metric that asserts correctness instead of measuring quality: hard-filter violation rate, which must be **zero**, not merely low.

### 6.2 Search invariants

| # | Invariant |
|---|---|
| S1 | **Constraints are never violated.** Every returned result satisfies every stated filter. The violation rate is zero, and this is asserted by test, not observed as a metric trend. |
| S2 | **Pagination is stable** under concurrent catalog writes (inherits C5). |
| S3 | **Degradation is announced, never silent.** If the query could not be served by the full pipeline, the response says so. |
| S4 | **Search availability never affects payment correctness.** A degraded or unavailable index cannot change whether a payment verifies or settles. |

### 6.3 The retrieval pipeline, sequenced

Ordering here is deliberate: each stage ships only after the previous one is measured, so every addition has a baseline to prove itself against.

| Stage | Mechanism | When |
|---|---|---|
| 0 | **Deterministic constraint filtering** over network, asset, type, price bounds, extensions | T#2, first |
| 1 | **Lexical retrieval** over the filtered set (Postgres full-text), plus a literal domain/URL-substring path for queries that *are* a hostname | T#2, first, and establishes the measured baseline |
| 2 | **Dense retrieval** fused with lexical, with the **lift over the baseline published** | Intended, and gated on the measured lift justifying its cost |
| n/a | *Neural reranker* | Out of scope for this grant, see §5.6 and below |

Stage 1 is fully functional with **no model dependency**, which is what keeps self-hosting inside the RFP's sub-one-hour target. The domain-substring path exists because a real agent query is sometimes just `api.example.com`, which stemmed full-text search handles poorly.

**On the reranker.** A cross-encoder offers the largest single gain in ranking quality, and we are leaving it out of this scope. The catalog stays sparse through the grant, so the gain would be small where we could measure it, and a model reading seller-supplied prose sits directly in the agent-facing path that §5.6 exists to constrain. Revisiting it once the catalog carries real volume is a reasonable next step, and the retrieval design leaves room for it.

### 6.4 What gets indexed, and what does not

Linking back to C2–C4 and §5.6:

- **Indexed text is a documented, deterministic function of specific structured seller fields**: not an opportunistic blend of whatever prose arrived. Reproducible input means a reproducible index and an auditable relevance story.
- **Facilitator-derived fields drive constraints**; seller-authored fields influence relevance only. Since price, asset, network, and owner come from the settlement, a seller cannot manipulate what they are *filtered* by, only how they are *described*.
- **Seller text carries untrusted provenance through the index and out to results**, so consumers can distinguish it structurally from settlement-derived facts.

### 6.5 Honest degradation

`/discovery/search` supports cursor pagination (S2) and reports two things most search APIs omit:

- **`partialResults`**: set when the pipeline could not complete: dense retrieval unavailable, a stage timed out, the index is mid-reload. The alternative is silently returning degraded results that look authoritative, which for an autonomous consumer is worse than an explicit partial.
- **`searchMethod`**: how this particular query was satisfied: filter-only, lexical, hybrid, or domain fallback. Useful to an agent deciding how much to trust an ordering, and useful to us as evaluation telemetry.

Both exist to serve S3. A degraded index degrades *discovery*, never payment (S4).

### 6.6 The evaluation harness

This is the graded deliverable. The RFP's evaluation criteria ask for a *search quality evaluation methodology*, not a sophisticated model. Metrics are chosen because each answers a distinct question:

| Metric | Question it answers |
|---|---|
| **Hard-filter violation rate** | Are we ever *wrong*? Must be zero (S1). |
| **nDCG@10** | Is the ordering good where an agent actually looks? |
| **MRR** | How quickly does the first genuinely useful result appear? |
| **Recall@20** | Did we surface the right resource at all, or lose it in retrieval? |
| **No-result accuracy** | When nothing matches, do we correctly return nothing instead of the closest plausible thing? A confidently wrong answer is expensive for an agent that pays per call. |
| **Latency p50/p95/p99, warm and cold** | Does it meet the RFP's fast-query requirement in the state operators actually run it? |

The harness runs in **CI as a regression gate** on every release, with the gate itself validated by a deliberately regressing commit, because a gate nobody has watched fail is not known to work. Results are published each tranche, **including negative ones**: if a stage does not earn its cost on our corpus, we say so with numbers.

### 6.7 Corpus and query set: the honest hard part

Automatic cataloging means the live catalog stays **sparse for the whole grant period**, since resources appear only as they are paid for. Any evaluation therefore rests on a constructed corpus, and the integrity of that construction is the thing worth engineering.

**Corpus composition**, with every record **source-marked**:
- realistic synthetic Stellar-priced resources across networks, assets, and price bands;
- **Stellar-specific fixtures**: SEP-41 asset variants, network variants, price-boundary cases;
- **adversarial fixtures**: near-duplicate listings, misleading descriptions, and prompt-injection attempts in seller text, so we measure §5.6's controls instead of asserting them.

**Metrics are sliced by record source**, and synthetic records are never presented as real. This is the discipline that keeps a constructed benchmark honest.

**Query set**: hand-written and reviewed, covering ordinary task queries, paraphrases, constraint-bearing queries, MCP-tool discovery, no-result cases, and adversarial cases. Split into a development set and a **held-out set that we do not tune against**, so reported numbers are not the product of fitting our own exam.

**Relevance judgments**, scoped honestly to the budget:
- **Hard constraints are never judged.** They are checkable facts, evaluated deterministically. No model or human opinion enters.
- **Semantic relevance is graded against a documented rubric** with reviewer agreement reported on a sample.
- We state our set sizes plainly and note the limitations. A smaller judged set, reproducibly built and honestly described, is worth more than a larger number we cannot stand behind, and the methodology is what the criteria reward.

## 7. MCP discovery interface

The RFP asks for an MCP server that lets an agent search the Stellar Bazaar and make a paid call **from inside its own runtime**, with structured inputs and outputs and machine-readable error codes. Two tools cover it: `search_resources` and `paid_call`.

### 7.1 The server holds no keys

This is the section's load-bearing decision. `paid_call` orchestrates the full loop: discover, receive the 402, obtain an authorization, settle, and retry the original request. **Signing is always delegated to the agent's own wallet or runtime.** The server never holds a signing key and never holds funds.

The reason is structural, not cautious. An MCP server that held agent keys would be a custody honeypot, and it would break the non-custodial guarantee the rest of the system is built on (§2.2): a compromised discovery service could then spend on behalf of every agent connected to it. Delegating signing means a compromised server can fail to serve, mislead about *which* resources exist, or refuse to settle. It cannot move anyone's money.

Both deployment shapes are supported by the same design:

- **Local (in-runtime) deployment**: signing happens in-process through the agent's signer.
- **Hosted deployment**: the server returns the material to be authorized, the client signs, and the signed authorization comes back. The extra round trip is the price of not holding the key, and it is worth paying.

### 7.2 One signing interface, shaped around auth entries

Because Stellar authorizes with Soroban authorization entries and not pre-signed transactions (§2.2), the signing interface is shaped around **authorizing an entry**, not around producing a transaction signature. That single shape covers both signer classes the RFP requires:

- a classic Ed25519 keypair, and
- a custom account / smart wallet whose `__check_auth` enforces its own policy.

Two reference signers ship, and both are exercised by the same conformance suite, so "we support smart accounts" is a tested claim, not a configuration option nobody ran.

### 7.3 Errors classified by what the agent should do next

The RFP requires machine-readable error codes and non-null reasons. For an autonomous consumer, the *cause* of a failure matters less than the **required response**, so one shared vocabulary spans discovery, verify, settle, and the MCP surface, and every code is classified:

| Class | Meaning for the agent | Examples |
|---|---|---|
| **Retryable** | Transient; the same request may succeed later | RPC outage, sequence contention, rate limit |
| **Remediable** | Fix a precondition, then retry | Missing or unauthorized trustline, insufficient balance, expired authorization |
| **Fatal** | Never retry this request as formed | Wrong asset, wrong network, unsupported scheme, ceiling exceeded |

An agent that cannot distinguish these three either gives up on transient failures or hammers a permanently invalid request. Classification is part of the contract, not documentation.

One shared vocabulary also means an agent learns a single error model for the whole system instead of four. Per §2.3, the seam guarantees every internal rejection maps onto a non-null protocol reason, so no path can silently lose its explanation.

### 7.4 Untrusted content crosses here

MCP is the delivery vehicle for the boundary §5.6 describes, since search results and tool responses carry seller-authored text straight into an agent's context. The controls therefore apply at this interface specifically:

- seller-authored fields are **provenance-marked as untrusted** and structurally separated from settlement-derived facts;
- seller text is **never placed in an instruction position** within a tool response;
- facilitator-derived fields (asset, network, settled price, owner) are the ones an agent can rely on, because they come from the settlement and not from a claim.

### 7.5 A retried call must not pay twice

Agents retry aggressively, often on timeouts that were actually successes. `paid_call` therefore carries an idempotency key from the caller, and it inherits the settlement-layer protection from §2.2: we persist the envelope and its hash before submission, and recovery polls by hash instead of resubmitting. A duplicated `paid_call` resolves to the original settlement and never produces a second one.

### 7.6 Spend control belongs to the signer

Two primitives already bound what an agent can spend, and both sit below this interface. The `upto` ceiling caps a single call, since the payer signs a maximum the settlement cannot exceed. A Soroban custom account enforces arbitrary policy inside `__check_auth`, covering per-period budgets, recipient allowlists, and velocity limits, and we support custom accounts as signers.

The MCP layer adds no cap of its own. It holds no keys and never observes payments an agent makes through any other path, so a limit enforced here would be advisory and an agent could bypass it by calling the resource directly. We do not present a convenience as a safety control.

### 7.7 Interface invariants

| # | Invariant |
|---|---|
| M1 | **The MCP server holds no signing keys and no funds.** Signing is always client-side. |
| M2 | **Every rejection carries a non-null, machine-readable reason**, classified as retryable, remediable, or fatal. |
| M3 | **Seller-supplied text never appears in instruction position** and always carries untrusted provenance. |
| M4 | **A retried `paid_call` cannot produce a second settlement.** |

## 8. Threat model and security properties

### 8.1 Method, and why it sits here

We authored this threat model **with** the architecture, not after it. That ordering is the point. A threat model written after the fact documents a system. One written alongside it *shapes* the system. Several decisions earlier in this document exist because of the analysis below: the contract never holding funds (§4.4), never dereferencing seller URLs (§5.5), leaving the reranker out of scope (§6.3), and the MCP server holding no keys (§7.1).

The discipline we hold ourselves to: **every surface maps to a control, and every control maps to a test.** A control asserted but never exercised is a claim, not a defence. The invariants named through this document, I1 to I11 for the `upto` contract, C1 to C5 for the catalog, S1 to S4 for search, and M1 to M4 for MCP, are the machine-checkable end of that mapping.

### 8.2 Assets

| Asset | Why an attacker wants it |
|---|---|
| **Sponsor XLM balance** | We pay network fees; every sponsored settlement spends our money |
| **Buyer funds in flight** | Value under authorization, mid-settlement |
| **Seller proceeds** | Settled funds en route to `payTo` |
| **Catalog integrity** | Whoever controls what agents can find, steers what agents buy |
| **Agent budgets** | Autonomous spend authority, exercisable without a human in the loop |
| **Operator keys** | Sponsor and channel accounts |
| **Service availability** | Denial is cheaper than theft and often sufficient |

### 8.3 Actors

Hostile buyer · hostile seller · hostile SEP-41 token contract · hostile custom account (`__check_auth`) · hostile listing *content* (targeting agents, not us) · compromised operator · degraded infrastructure · network observer.

The distinction that shapes the design: **the fee sponsor makes us a spending party in every transaction.** Most facilitator threat models treat theft as the goal. Here, an attacker who can make us *pay* without stealing anything has already succeeded, so fee-drain paths get first-class treatment.

### 8.4 Settlement surfaces

| # | Threat | Control | Test / invariant |
|---|---|---|---|
| T1 | **Replay** of a signed authorization | Single-use via host nonce; replay state checked *before* any fee is spent; submission idempotency by persisted hash (§2.2) | I5; duplicate-submission test |
| T2 | **Fee drain via `__check_auth`** | Dual resource + inclusion fee ceilings applied at the enforcing-simulation gate, before fee commitment; per-caller sponsor budgets | CPU-burning custom-account double (§4.5) |
| T3 | **Fee drain via pathological token contract** | Mainnet asset allowlist (§2.2); fee ceilings | Adversarial SEP-41 doubles (§4.5) |
| T4 | **Fee drain via mass failing settlements** | Enforcing simulation rejects *before* submission; caller metering and rate limits; failed-settlement fee-spend alarm | Negative matrix; monitored signal (§9) |
| T5 | **Verification bypass via simulation confusion**, treating a passing record simulation as proof of authorization | Signature and authorization validation are explicit and independent of simulation | Case where record simulation succeeds but the signature is invalid must reject |
| T6 | **Fund redirection** | Recipient bound in signed material; facilitator never appears as `from`, source, or payment signer; startup refuses a fund-moving key | I6; wrong-recipient rejection |
| T7 | **Settling more than authorized** | `0 ≤ actual ≤ max` asserted in-contract; exact-amount binding for `exact` | I3; above-max rejection |
| T8 | **Sequence exhaustion under burst** | Fenced channel-account leases; concurrency and rate limits; explicit `TRY_AGAIN_LATER` handling | Load test asserting zero duplicate settlements |
| T9 | **Front-running / authorization griefing**, a third party burning the payer's authorization | The settle call is bound by the facilitator's own `require_auth`, so a third party cannot invoke it; bindings make early submission non-profitable | I10; unauthorized-invoker rejection |
| T10 | **Rent inflation via long signature expiry** | Three-clock ordering derived from a bounded `maxTimeoutSeconds` (§4.6) | I8; clock boundary tests at before / equal / after |

### 8.5 Catalog surfaces

| # | Threat | Control | Test / invariant |
|---|---|---|---|
| T11 | **Listing spoofing**, claiming another party's resource | Owner is the settled recipient, never client-supplied | C2; mismatched-`payTo` case |
| T12 | **Price spoofing**, advertise cheap and charge more | Price, asset, network derived from the settlement | C3; metadata/settlement mismatch case |
| T13 | **`routeTemplate` traversal** | Percent-decode before traversal checks, bounded repeat-decoding | Multi-layer encoded traversal fixtures |
| T14 | **SSRF via seller-supplied URLs** | Never dereferenced server-side, eliminated as a category and not filtered | C4; assertion of no outbound fetch of seller values |
| T15 | **Catalog flooding** | Economic floor (a listing costs a settled payment) plus per-recipient insert limits and metadata size bounds | Flood simulation |
| T16 | **Listing takeover on contested identity** | Quarantine for review; never last-write-wins | Contested-identity case (§5.3) |
| T17 | **Weaponised metadata to break payments** | Cataloging is strictly subordinate to settlement | C1; invalid-metadata-with-valid-payment case |

### 8.6 Agent-facing surfaces

The surface we consider most under-examined in this field, and the reason we staff AI-security specialists instead of consulting them at the end.

| # | Threat | Control | Test / invariant |
|---|---|---|---|
| T18 | **Prompt injection via seller metadata** into agent context | Structural separation; untrusted provenance carried end-to-end; seller text never in instruction position; facilitator-derived fields drive constraints | M3; injection fixtures in the evaluation corpus (§6.7) plus MCP response assertions |
| T19 | **Induced overspend**, a listing luring an agent into expensive calls | `upto` ceiling bounds a call; custom-account policy can bound a signer | I3; residual risk acknowledged (§8.8) |
| T20 | **Search-result manipulation** | Constraints are facilitator-derived and unforgeable; seller prose influences relevance only; volume-based quality signals not used during the grant | S1; hard-filter violation rate of zero |
| T21 | **Key exposure at the MCP server** | The server holds no keys, structurally | M1; source-level assertion in conformance |
| T22 | **Double payment via agent retry** | Caller idempotency key plus settlement-layer hash idempotency | M4; retried `paid_call` case |

### 8.7 Operator and infrastructure surfaces

| # | Threat | Control | Test / invariant |
|---|---|---|---|
| T23 | **Compromised operator** | Every settlement is bound to the payer's signature; non-custody enforced at startup | See the boundary statement below |
| T24 | **Sponsor / channel key exposure** | KMS or hardware-backed storage; rotation runbook; startup refuses to serve mainnet without correctly scoped keys | Startup-gate test |
| T25 | **Degraded or lying RPC** | Two independent providers with failover; recovery by hash; never blind resubmission | Failure-injection matrix |
| T26 | **Dependency compromise** | Lockfiles, vulnerability scanning, and a licence gate in CI (also serving the permissive-OSI requirement) | CI gate |
| T27 | **TTL / rent lapse on the `upto` contract**, instance or Wasm archived and settlement unavailable | TTL monitoring with restore-before-archival alerting; documented restore procedure | Monitored signal (§9) |

**What a compromised operator can and cannot do.** Stating this precisely matters more than asserting trustworthiness. A malicious or compromised facilitator **can** refuse service, censor listings, misreport what exists in the catalog, and degrade discovery quality. It **cannot** move user funds, redirect a payment to a different recipient, settle more than was authorized, or forge a settlement, because every settlement is cryptographically bound to the payer's own authorization and every settled transaction is publicly verifiable on-chain. The residual risks are availability and honesty-of-discovery, and the structural mitigation is that the whole system is permissively licensed and self-hostable: anyone can run their own and stop trusting us.

### 8.8 Residual risks, stated plainly

- **Metering honesty is outside the scheme.** `upto` bounds what a seller *can* collect; it cannot prove the seller measured usage honestly. A seller may always charge the ceiling. This is an inherent property of metered billing, not a defect we can engineer away, and buyers should treat the ceiling as the real price when choosing a counterparty.
- **Discovery censorship and misreporting** by the hosted operator remain possible (T23), mitigated only by self-hostability.
- **Relevance gaming** is partially achievable by a seller writing better prose. Constraints are not gameable; ordering is.
- **Prompt injection can be constrained, not eliminated.** We bound where untrusted text may appear; we do not claim to detect adversarial natural language.
- **Clawback-enabled assets** allow an issuer to reverse a settled payment. We surface the asset property; we cannot prevent the behaviour.
- **Induced overspend** beyond the ceiling and signer-policy primitives (T19), pending the §7.7 decision on whether the MCP layer should carry its own caps.

### 8.9 The AI-security engagement

Time-boxed and scoped to the agent-facing boundary: the path from seller-supplied metadata, through the catalog and search, into MCP tool responses and an agent's context.

We treat this as an **investigation, not a checklist**, because the area is new enough that we cannot say in advance which findings will reduce to mechanical checks. Some will. A control on where untrusted text may appear in a tool response is testable. Others will be behavioural observations about how an agent responds to adversarial content, which is not a deterministic property of our system. Committing in advance to convert every finding into a CI test would therefore be a commitment we could not honestly keep.

The output is a **published assessment** stating scope, method, findings, and limitations. Each finding is accompanied either by a test, where it reduces to a deterministic check, or by documented design guidance with the reasoning for why it does not. The assessment is written to be usable by other teams building x402 and Bazaar implementations on Stellar, not only by us.

## 9. Operations: configuration, monitoring, self-hosting

### 9.1 Asymmetric posture: testnet permissive, mainnet fail-closed

The RFP asks for two different things from the two networks: testnet *free and usable*, mainnet *safe and configurable*. The operational posture is therefore deliberately asymmetric.

**Testnet** boots with no API key, no external accounts, and no paid dependencies, and it stays free. Coarse rate limiting is the only gate. This is what makes the sub-one-hour onboarding claim in §9.6 achievable at all.

**Mainnet fails closed.** The service refuses to serve mainnet traffic unless every one of the following holds:

| Gate | Why |
|---|---|
| Sponsor key present and correctly scoped | Sponsorship requires a funded key; a *fund-holding* key in that slot would break non-custody, so startup rejects it |
| Fee ceilings configured, from measured values | An unmeasured ceiling is an unbounded fee-drain exposure (T2–T4) |
| Asset allowlist non-empty | An open mainnet asset set is a fee-drain vector (§2.2) |
| Settlement contract address configured **and its on-chain Wasm hash matching the expected build** | The contract is immutable and gets replaced, never upgraded (§4.2). This gate is what prevents pointing at a substituted contract |
| Channel pool provisioned | Sequence contention under burst degrades into duplicate-submission risk (T8) |
| At least one RPC provider reachable | Fail at startup, not mid-settlement |

### 9.2 Caller authentication, metering, and the fee-drain control

Required to be documented by the RFP, and shaped by the fact that we spend money on every sponsored settlement.

Mainnet callers authenticate with API keys carrying per-key metering, rate limits, and concurrency limits. The substantive question is how sponsored fee spend is bounded, and there are two options with a real trade-off:

- **Prepaid per-submission debit** is the stronger control, since a caller cannot spend past a balance they funded, but it puts an onboarding step in front of every integrator.
- **Post-hoc metering** is frictionless but lets a burst overspend before anyone notices.

Our opening approach is a **fixed daily budget for sponsored transaction fees**, enforced before submission. Total sponsored spend cannot exceed that budget in a day, whatever the mix of callers, which bounds our exposure to a known number from the first day of mainnet operation without putting an onboarding step in front of any integrator.

We treat the optimal control as an open research question for development time, not something to settle in advance. Once we have measured real caller behaviour and real settlement cost on testnet and then pubnet, we will evaluate per-key budgets, prepaid per-submission debit, and combinations of the two against what abuse actually looks like, and we will publish what we choose and why. The fixed daily budget is the floor that holds while that work happens.

**Mainnet pricing is operator configuration**, not our business model. It is a facilitator fee that an operator may set, change, or remove entirely, stated in public configuration. Self-hosters set their own or none. Whatever a deployment charges is visible, and never taken as a spread.

### 9.3 The three coupled controls

`maxTimeoutSeconds`, the fee ceilings, and signature expiry are **one control with three settings**, documented and configured together. They interact: a long validity window inflates rent on nonce and allowance state (T10) while a short one causes spurious rejections under load, and both interact with the ceiling that determines whether an expensive `__check_auth` is affordable. Presenting them as three independent knobs is how operators misconfigure this class of system, so we do not.

### 9.4 Monitoring, with thresholds we derive

Signals are grouped by what they protect. The plan ships with a **measurement procedure**, and we populate thresholds from measured p99 cost on testnet, because a threshold someone picked from intuition is not a control.

| Group | Signals |
|---|---|
| **Sponsor solvency** | XLM balance, burn rate, fee spend per settlement, **failed-settlement fee spend** (the drain alarm), per-key budget consumption |
| **Settlement health** | verify/settle volume, success and failure ratio, latency p50/p95/p99, rejection reasons broken down by class |
| **Channel pool** *(supporting)* | utilisation, lease contention, sequence-collision rate, `TRY_AGAIN_LATER` frequency |
| **Contract liveness** | instance TTL and Wasm code TTL countdowns, with restore-before-archival alerting (T27) |
| **Catalog** | cataloging lag, soft-drop rate by reason, per-recipient insert rate for flood detection, quarantine depth |
| **Discovery** | query latency, zero-result rate, and `searchMethod` distribution, where a shift is how we learn retrieval has silently degraded |
| **Infrastructure** *(supporting)* | per-provider RPC health, failover events, datastore health |

### 9.5 RPC failover that cannot double-settle

Two independent Soroban RPC providers with failover on error or timeout. The constraint that shapes the implementation: **failover reads, it never re-submits.** A lost response is resolved by polling for the persisted transaction hash (§2.2), never by submitting again, so a provider switch mid-settlement cannot produce a second payment (T25).

### 9.6 Self-hosting and the sub-one-hour path

The RFP requires that a developer reach a discoverable, paid endpoint "in well under an hour." That is a measurable claim, and several earlier decisions exist to make it true:

- **Single-command bring-up**, containerised, reproducible from a clean clone.
- **Testnet needs zero external accounts**: no API keys, no model weights, no paid services. This is the operational payoff of shipping lexical search with no model dependency (§6.3) and no reranker: the whole stack runs offline on a laptop.
- **The seller path is short by design**: an unmodified endpoint, declare metadata, take one payment, appear in the catalog, with `validate`-without-paying (§5.9) so a metadata mistake costs a command instead of a settlement.
- **Timed onboarding is an acceptance criterion**, measured on a clean machine, and it is exactly what the professional user testing in T#3 exists to evidence.

### 9.7 Availability, stated as two numbers

A single uptime figure would hide the criticality layering this architecture depends on, so the SLO distinguishes:

- **Payment-path availability**: verify and settle. Target 99%+ over a rolling 30-day window.
- **Discovery availability**: catalog and search. Reported separately, because degraded discovery must never imply degraded payment correctness (S4).

Exclusions are stated up front: planned maintenance, and upstream Stellar or RPC-provider outages we do not control. Publishing a number with unstated exclusions is worse than publishing none.

### 9.8 Runbook derived from the threat model

Every threat in §8 that can manifest operationally has a runbook entry: sponsor drain, key compromise and rotation, RPC outage and failover, contract TTL approaching archival, catalog flooding, contested-identity adjudication, degraded-mode operation, and rollback to a previous release. Deriving the runbook from the threat model, instead of writing it independently, gives a closure property worth having: **there is no incident class we analysed but did not prepare for.**

### 9.9 Maintenance and specification drift

The RFP requires a stated post-grant conformance maintenance plan, and the Bazaar extension is a moving target (§5.10). What we commit to:

- the conformance suite pinned to an explicit specification version, re-run against both networks on a stated cadence;
- specification changes tracked as issues with a documented triage path;
- **named maintainers**, not an unattributed commitment.

We will also be explicit about the boundary: we commit to routine conformance maintenance and security patching post-grant, because the project is load-bearing for work we intend to continue. A major protocol revision requiring substantial re-engineering gets scoped separately, and we do not silently assume it.

