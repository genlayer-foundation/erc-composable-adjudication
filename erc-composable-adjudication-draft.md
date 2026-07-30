# ERC-XXXX: Composable Adjudication (Draft v0.1)

> Natural-language draft. No Solidity yet — function names and types are given for precision, but the normative content is the prose. Status: pre-draft.

## Abstract

This ERC defines a minimal standard interface for on-chain **adjudication**: submitting a case for decision and consuming its resolution. A contract implementing the interface is an **Adjudicator**; a contract that registers adjudications and acts on their resolutions is an **Adjudicable**. The interface is deliberately small — a two-phase lifecycle in which an adjudication is **registered** when parties commit to it (typically at agreement time, before any contest) and **activated** only if a decision is actually needed, plus a pull-based resolution — with all case semantics (what is being decided, the evidence, the meaning of the outcome) carried as Ethereum Attestation Service (EAS) attestation pointers rather than on-chain structures. Because the interface makes no assumptions about *how* decisions are reached, an aggregate of Adjudicators — an escalation chain, an X-of-Y panel — can itself implement the interface, making adjudication composable. Composition interfaces and ready-made composites are specified in a companion ERC.

## Motivation

Adjudication systems in production today — GenLayer, Kleros, UMA, and single off-chain agents operating a wallet — each expose incompatible interfaces. Consequences:

1. **Integration cost.** A protocol needing both deterministic and subjective resolutions implements several bespoke integrations.
2. **No composition.** There is no standard way to combine systems: escalation chains or agreement across a panel of independent systems.
3. **Lock-in.** Choosing an adjudication system at integration time is effectively permanent.

Prior standardization attempts inform this design. ERC-792 standardized a two-sided arbitrator/arbitrable pair with callback-based delivery; its evidence companion ERC-1497 saw implementations deviate significantly from the specification. This ERC takes the lessons: standardize one side only, standardize state rather than behavior, and push everything semantic to an attestation layer that can evolve without touching deployed contracts.

## Specification

The key words MUST, MUST NOT, SHOULD, and MAY are to be interpreted as described in RFC 2119.

### Roles

- **Adjudicator** — a contract implementing this interface. It accepts adjudications and eventually resolves them. It may be a native system, a thin adapter in front of an existing system (GenLayer, Kleros, UMA), or a composite of other Adjudicators.
- **Adjudicable** — any contract or account that registers an adjudication and consumes its resolution. This ERC imposes **no interface** on Adjudicables.

### Core interface (normative, in words)

An Adjudicator MUST expose:

1. **`registerAdjudication(bytes32 definitionUID, bytes data) payable returns (uint256 adjudicationId)`**
   Puts a new adjudication on the books **without starting any adjudication work** — the on-chain equivalent of an arbitration clause, intended to be callable at agreement time, before any contest exists.
   - `definitionUID` — EAS attestation UID of the **adjudication definition**: what is being decided, the possible outcomes and their meaning, and any process parameters. The definition carries or references the entire baseline (the parties' agreement and its context), so registration needs no evidence parameter — evidence, as an interface concept, exists only to argue a contest. MAY be zero, meaning the Adjudicator itself supplies the definition (a template Adjudicator with its definition fixed at deployment); implementations that require a caller-supplied definition MUST revert on zero. The `AdjudicationRegistered` event MUST carry the definition UID actually governing the case — a template Adjudicator emits its fixed definition's UID, never zero — so indexers see a uniform stream.
   - `data` — implementation-defined extra call parameters. MAY be empty. (Semantic content never travels here — it lives behind EAS pointers.)
   - The function is `payable`. Whether payment is required is **implementation-defined**, but registration SHOULD be cheap or free — adjudication fees belong to activation.
   - `adjudicationId` MUST be unique within the Adjudicator and MUST NOT ever be reused. Implementations MUST NOT assign sequential or otherwise predictable identifiers: ids are drawn unpredictably (e.g., a hash over registrant, `definitionUID`, and a salt), so an id cannot be predicted, squatted, or front-run before its registration lands. Registration MUST NOT fail because the same `definitionUID` was registered before — one definition can serve many cases.
   - On success the Adjudicator MUST emit `AdjudicationRegistered` and the adjudication's status MUST become `Registered`.

2. **`activate(uint256 adjudicationId, bytes32[] evidenceUIDs, bytes data) payable`**
   Starts adjudication of a registered case — the moment a contest actually exists.
   - MUST revert unless status is `Registered`.
   - `evidenceUIDs` — EAS attestation UIDs of the activator's contest evidence, linked atomically with activation (each one MUST emit `EvidenceLinked`). MAY be empty — some contests are self-evident given the baseline.
   - `data` — implementation-defined extra call parameters, mirroring registration's `data` (e.g., parameters an adapter needs at activation, when it starts the underlying system's own process; relayed opaquely by composites). MAY be empty; never semantic content.
   - **Authorization is governed by the adjudication definition — and enforceable, because definitions are on-chain attestations the contract can read.** Where the definition names the case's parties, the Adjudicator SHOULD restrict `activate` to exactly those parties (each unilaterally — a contested counterparty's cooperation cannot be required); registering a case confers no activation rights. Open activation by anyone is legitimate only where the definition explicitly opts into it (e.g., oracle-style questions). Only where the definition names no parties and does not opt into open activation SHOULD implementations fall back to restricting activation to the registrant.
   - The function is `payable`; this is typically where adjudication fees are due. Amounts, assets, and refund behavior are **implementation-defined**; the Adjudicator MUST revert if its payment requirements are not met.
   - On success the Adjudicator MUST emit `AdjudicationActivated` and status MUST become `Active`. Because the activator controls activation-time evidence, implementations SHOULD give the case's parties an opportunity to link evidence after activation.

3. **`status(uint256 adjudicationId) returns (Status)`** where `Status` is the enum **`None / Registered / Active / Resolved`**:
   - `None` — no adjudication exists under this id. (This value MUST be the enum's zero value: an unwritten storage slot reads as zero, so zero must mean "does not exist.")
   - `Registered` — the case is on the books, binding and referenceable, but dormant: no adjudication work is happening.
   - `Active` — adjudication is underway with no final resolution yet. All intermediate conditions (evidence gathering, deliberation, internal rounds, awaiting escalation inside a composite) map to `Active`.
   - `Resolved` — a final resolution exists. Status MUST NOT change once `Resolved`.

4. **`resolution(uint256 adjudicationId) returns (bytes32)`** — the EAS attestation UID of the **resolution**. MUST return the UID once status is `Resolved` and MUST revert or return zero before that. Once `Resolved`, the value returned MUST equal the `resolutionUID` emitted in the adjudication's `Adjudicated` event and MUST NOT ever change. The ERC does not constrain the resolution's content; its schema is referenced by the adjudication definition.

5. **ERC-165.** Every Adjudicator MUST implement `supportsInterface` and MUST answer `true` for this interface's ID. This is REQUIRED (not optional): it is the discovery mechanism for the interface itself and the rail on which all future extensions ride.

### Events (normative)

- **`AdjudicationRegistered(uint256 indexed adjudicationId, address indexed registrant, bytes32 definitionUID)`** — MUST be emitted on registration.
- **`AdjudicationActivated(uint256 indexed adjudicationId, address indexed activator)`** — MUST be emitted on activation.
- **`EvidenceLinked(uint256 indexed adjudicationId, bytes32 evidenceUID, address indexed submitter)`** — MUST be emitted for each evidence UID linked at activation, and for any evidence linked later if the implementation allows it.
- **`Adjudicated(uint256 indexed adjudicationId, bytes32 resolutionUID)`** — MUST be emitted exactly once, when the adjudication becomes `Resolved`.

### Lifecycle

`None → Registered → Active → Resolved`, forward-only, each transition once and irreversibly. **No state may be skipped**: an adjudication MUST pass through `Registered` and `Active` before it can be `Resolved`. A single transaction MAY perform several transitions back-to-back (e.g., register-and-activate, or a composite's uncontested path activating and resolving in one call), but each transition MUST occur and MUST emit its event. Nothing else is standardized. In particular this ERC defines **no** appeal, escalation, or challenge entry point, and no state to represent them: an Adjudicator with internal appeal rounds, or a composite awaiting escalation, simply remains `Active` until its outcome is final.

### Resolution delivery: pull only

The Adjudicator MUST NOT be required to call into any consumer. Consumers observe the `Adjudicated` event and read `resolution(adjudicationId)`. Adjudication can therefore never be blocked by a failing consumer, and no consumer-side interface exists to standardize or to deviate from.

### EAS as the semantic layer

Three kinds of attestations, whose schemas are registered in an EAS schema registry:

- **Adjudication definition** — the question, the outcome space and its meaning, and process parameters (e.g., evidence windows, escalation conditions, who may activate). It carries or references the **baseline**: the parties' agreement and its context, committed while the parties still cooperate — which is what makes the baseline tamper-proof at contest time. Created before or at registration.
- **Evidence** — contest material: content or content pointers, following the evidence schema, brought to argue an activated case. Linked at activation; later linking is implementation-defined but MUST emit `EvidenceLinked`. **The normative link between evidence and a case is the on-chain `EvidenceLinked` event.** An evidence attestation SHOULD additionally reference the definition via EAS `refUID` for off-chain traceability, but `refUID` alone is not sufficient to bind evidence to a case: one definition can serve many adjudications (template adjudicators), so only the event is unambiguous.
- **Resolution** — the outcome, referencing the definition it answers.

**Attestation integrity (normative).** The lifecycle's finality guarantee is only as strong as the content behind the UIDs, so:

- Definition and resolution attestations MUST be irrevocable: their EAS schemas MUST be registered with revocability disabled. A resolution whose content could be revoked after `Resolved` — leaving consumers holding an immutable pointer to a repudiated statement — would defeat the standard's load-bearing guarantee.
- Definition and resolution attestations MUST have an `expirationTime` of zero. An expiring attestation is revocation on a timer: a resolution that expires self-repudiates on schedule under EAS validity semantics despite passing every other check, and a definition that expires during a long `Registered` dormancy evaporates the baseline exactly when a contest finally needs it.
- Resolution attestations MUST be on-chain attestations.
- Definition attestations MUST be on-chain attestations on the same chain as the Adjudicator, so their content is readable from the contract — this is what makes definition-governed rules (such as activation authorization) enforceable rather than advisory. Definitions SHOULD stay compact, embedding content hashes for bulky baseline material rather than mutable references (such as URLs), so that substitution or withholding of the baseline is detectable.

### Composition (informative — normative content in the companion ERC)

Because composites implement this same interface, a consumer cannot and need not distinguish a single system from an escalation chain (e.g., GenLayer → Kleros, or ending in an off-chain ADR body behind an adapter) or an X-of-Y panel (2-of-3 across GenLayer, Kleros, UMA; 11 off-chain agents by majority). Internally a composite registers and activates child adjudications on other Adjudicators through the same standard surface; a "round" is nothing more than a child adjudication. The introspection interface for walking a composite's children, the delegation event, escalation entry points, and their funding patterns are specified in the companion Composition & Factories ERC, together with factories that instantiate common composites pre-wired.

### Chain deployment (informative)

The interface is chain-agnostic and expected to be deployed on many EVM chains. Where an Adjudicator fronts a system living elsewhere — another chain, an L2, or off-chain infrastructure — cross-chain and off-chain mechanics (messages carrying value, proxy pairs, relayer models) are entirely an adapter concern, invisible to this standard.

## Rationale

- **A new interface from zero.** This design is not derived from ERC-792, which serves as prior art only; it does not carry over the two-sided interface or callback semantics. Existing systems are bridged by adapters, which they need anyway to normalize their differing models.
- **Standardize state and pointers, not behavior.** ERC-1497's fate showed that over-specifying semantics invites deviation. Everything semantic lives in EAS attestations, which can evolve per use case without redeploying or re-standardizing.
- **Two phases: registration, then activation.** Real agreements fix their forum at signing and invoke it only on contest; the split mirrors that. Registering at agreement time — while parties still cooperate — establishes one canonical `adjudicationId` from day one, eliminating the competing-cases problem for agreements with no on-chain deal contract, and puts the commitment on the books at near-zero cost while adjudication fees wait until a contest actually exists. The naming is deliberately symmetric: `register → Registered`, `activate → Active`, resolve → `Resolved` — each state is the past participle of the verb that reaches it.
- **Exactly four statuses.** Any richer enum either enumerates mechanisms (appeal, escalation — a closed set that new systems would not fit) or duplicates information already available from the resolution itself. `None` is forced by EVM storage semantics: an unwritten slot reads as the enum's zero value, so zero must mean "does not exist" or nonexistent cases would be indistinguishable from registered ones.
- **Pull-only delivery.** Callback delivery lets a reverting consumer block resolution permanently — a known failure mode of callback-based arbitration — and requires standardizing a second interface. Pull delivery has neither problem.
- **`payable` registration and activation with implementation-defined fee semantics.** Some systems (Kleros) require payment in the same transaction that starts their underlying case — for an adapter, that transaction is activation — so the payment *channel* must be standard or adapters are impossible; fee *semantics* (amounts, assets, refunds, who pays) differ so much across systems that standardizing them would be false precision. Multi-stage funding follows the same logic: activation funds the first round; later rounds are funded through the composite's own entry points when — and only if — they are needed.
- **ERC-165 required.** Ten lines of boilerplate buy unambiguous discovery today and automatic detectability for every future extension.

## Possible future extensions (explicitly not specified now)

- **`Provisional` status + `challenge()`** — a fourth status signaling "a provisional resolution exists and an action window is open," with a standard challenge entry point. Deferred: challenge actions are inseparable from fees, which are out of scope.
- **`cost()` fee-query view** — a standard way to ask what `registerAdjudication` or `activate` requires. Deferred; callers learn costs from the adjudication definition or off-chain.
- **Capability introspection** — enumerating available actions per adjudication.
- **Consumer callback** — optional push notification, best-effort, never blocking resolution.

## Security considerations

Resolution finality is the load-bearing guarantee: once `Resolved`, status and resolution UID are immutable — and because resolution attestations are required to be irrevocable and on-chain (see *Attestation integrity*), the content behind the UID is immutable too — so consumers can safely act on them. Trust in *what* the resolution says is exactly trust in the chosen Adjudicator — the standard makes the choice explicit and swappable but does not reduce it. Registration is permissionless in principle, so duplicate or orphan registrations can exist; they bind nobody, because an adjudication only binds whoever committed to its specific `adjudicationId` in advance — an Adjudicable that registered the case itself and stored the returned id, or parties who registered during negotiation and referenced the id in their signed agreement. Since that commitment happens while the participants still cooperate, exactly one canonical case exists per agreement, and a case registered by anyone else is an orphan no consumer will ever read.

**Activation as an attack surface.** A malicious activator cannot forge an outcome — resolutions come from the Adjudicator, and a baseless contest resolves against the activator at the cost of the fee they paid. The remaining vectors are: *paid griefing* (activating a case with no genuine contest to drag the parties into a process — bounded, since the attacker pays full adjudication fees to inflict inconvenience) and *evidence capture* (an unauthorized or front-running activator supplying misleading activation-time evidence and hoping for a default outcome before the real parties respond). Both are addressed by the normative guidance above: activation authorization is governed by the definition and SHOULD default to the case's parties, which eliminates third-party activation entirely for two-party agreements; parties SHOULD be able to link evidence after activation, which defeats evidence capture; and `AdjudicationActivated` is an indexed event, so parties — or their monitoring agents — are expected to watch their registered cases rather than assume silence means safety.

**Liveness — withheld outcomes.** The guarantees above cover forged outcomes; they do not cover outcomes that never arrive. The lifecycle is forward-only with no terminal failure state, no timeout, and pull-only delivery, so nothing in this standard ever forces an Adjudicator out of `Active`. Concretely: an adjudicator whose resolving key is lost — or whose operator withholds the resolution to extort the parties — leaves every dependent case `Active` forever; parties who settle amicably after activation have no exit even by unanimous consent, since only the Adjudicator can reach `Resolved`; and a composite's liveness is the minimum of its children's, not the majority's — an X-of-Y panel with one permanently stalled child it still needs is stalled itself. **Resolution liveness is therefore part of the trust placed in the chosen Adjudicator**, exactly like resolution honesty, and choosing an Adjudicator means accepting its liveness assumptions. Accordingly: Adjudicables SHOULD NOT make irreversible commitments whose only exit is `Resolved`, and SHOULD implement a fallback for an Adjudicator that never resolves — a deadline after which a default path applies, an alternate resolution route, or a mutual-release mechanism the parties can invoke by consent. Adjudicator implementations and composites are free to add their own liveness mechanisms (operator timeouts, panels that tolerate stalled children); those are implementation concerns outside this standard.

Adapters fronting other chains or off-chain systems add their own trust assumptions (authorized submitter wallets, bridges); these belong to the adapter's documentation, not this standard.

## Prior art

- ERC-792, Arbitration Standard: https://docs.kleros.io/developer/arbitration-development/erc-792-arbitration-standard
- ERC-1497, Evidence Standard: https://docs.kleros.io/developer/arbitration-development/erc-1497-evidence-standard
- Kleros v2 arbitration interface, dispute templates, and evidence format: https://kleros.mintlify.app/developers/arbitrable-apps/arbitrable-guide
- ERC-8033: https://eips.ethereum.org/EIPS/eip-8033
