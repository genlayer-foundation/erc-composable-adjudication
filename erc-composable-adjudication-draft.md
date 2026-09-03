---
title: Composable Adjudication
description: Interface for registering and resolving adjudications, where a composite of adjudicators is itself an adjudicator
author: Iván Raskovsky (@rasca) <raskovsky@gmail.com>
status: Draft
type: Standards Track
category: ERC
created: 2026-08-04
requires: 165
---

## Abstract

This ERC defines a minimal standard interface for on-chain **adjudication**:
submitting a question for decision and consuming its resolution. A contract
implementing the interface is an **Adjudicator**; a contract that registers
adjudications and acts on their resolutions is an **Adjudicable**.

It consists of a three-state lifecycle, in which an adjudication is
**registered** ahead of any need for a decision, **activated** when its
activating condition occurs, and **resolved** when its outcome is final. All
semantic content (what is being decided, the evidence, the meaning of the
outcome) is carried as Ethereum Attestation Service (EAS) attestation pointers
rather than on-chain structures.

Because the interface makes no assumptions about *how* decisions are reached, an
aggregate of Adjudicators, such as an escalation chain or an X-of-Y panel, can
itself implement the interface, making adjudication composable. Composition
interfaces and ready-made composites are specified in a companion ERC.

## Motivation

Adjudication systems in production today (GenLayer, Kleros, UMA, and single
off-chain agents) each expose incompatible interfaces. Consequences:

1. **Integration cost.** A protocol needing both deterministic and subjective
   resolutions implements several bespoke integrations.
2. **No composition.** There is no standard way to combine systems: escalation
   chains or agreement across a panel of independent systems.
3. **Lock-in.** Choosing an adjudication system at integration time is
   effectively permanent.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Roles

- **Adjudicator**: a contract implementing this interface. It accepts
  adjudications and eventually resolves them. It may be a native system, a thin
  adapter in front of an existing system (GenLayer, Kleros, UMA), or a composite
  of other Adjudicators.
- **Adjudicable**: any contract or account that registers an adjudication and
  consumes its resolution. This ERC imposes **no interface** on Adjudicables.

### Core interface (normative, in words)

An Adjudicator MUST expose:

1. **`registerAdjudication(bytes32 definitionUID, bytes data) payable returns (uint256 adjudicationId)`**
   Records a new adjudication **without starting any adjudication work**. It
   fixes the forum and the question ahead of the activating condition, and is
   intended to be callable at commitment time, before any decision is needed.
   - `definitionUID`: the EAS attestation UID of the **adjudication
     definition**. The definition states what is being decided, the possible
     outcomes and their meaning, and any process parameters. It also carries or
     references the entire baseline: the agreement, the question, or the state
     of the world that the adjudication is decided against. Evidence, as an
     interface concept, exists only to argue an activated adjudication, so
     registration takes no such parameter.
   - **`definitionUID` MAY be zero, meaning the Adjudicator supplies the
     definition itself.** Such an Adjudicator is a template Adjudicator: its
     definition is fixed at deployment and governs every adjudication it
     registers. An Adjudicator that expects the caller to supply a definition
     MUST revert on zero.
   - **A template Adjudicator MUST revert on a nonzero `definitionUID` that does
     not equal its fixed definition's UID.** With the rule above, this means a
     caller can never register under definition X while the adjudication is
     governed by definition Y.
   - The `AdjudicationRegistered` event MUST carry the definition UID that
     actually governs the adjudication. A template Adjudicator therefore emits
     its fixed definition's UID and never zero, so indexers see a uniform
     stream.
   - `data`: implementation-defined extra call parameters. MAY be empty.
     Semantic content never travels here; it lives behind EAS pointers.
   - The function is `payable`. Whether payment is required is
     **implementation-defined**, but registration SHOULD be cheap or free;
     adjudication fees belong to activation.
   - `adjudicationId` MUST be unique within the Adjudicator and MUST NOT ever be
     reused. An id MUST depend only on its own registration's inputs, never on
     other registrations or on shared mutable state such as a counter: no third
     party's registration can change the id another registration receives, and
     sequential assignment is therefore non-conforming. Deterministic derivation
     from the registration's inputs (e.g., a hash over registrant,
     `definitionUID`, and a caller-supplied salt) is RECOMMENDED: it lets the id
     be precomputed and referenced off-chain before registration lands, and
     binding the id to the registrant means no one else can occupy it. Under
     deterministic derivation the input tuple MUST be unique per registration
     (the salt, or an equivalent input, provides this); if a registration would
     derive an id that already exists, the Adjudicator MUST revert rather than
     reuse or overwrite it. Registration MUST NOT fail merely because the same
     `definitionUID` was registered before; one definition serves many
     adjudications, distinguished by salt.
   - On success the Adjudicator MUST emit `AdjudicationRegistered` and the
     adjudication's status MUST become `Registered`.

2. **`activate(uint256 adjudicationId, bytes32[] evidenceUIDs, bytes data) payable`**
   Begins work on a registered adjudication at the moment its activating
   condition occurs and a decision is actually needed.
   - MUST revert unless status is `Registered`.
   - `evidenceUIDs`: EAS attestation UIDs of the evidence the activator brings,
     linked atomically with activation (each one MUST emit `EvidenceLinked`).
     MAY be empty; some adjudications need nothing beyond the baseline. The
     standard imposes no validity requirement on evidence UIDs (nothing requires
     a UID to reference an existing attestation); whether and how evidence UIDs
     are validated is implementation-defined, and divergence here is conforming.
   - `data`: implementation-defined extra call parameters, mirroring
     registration's `data` (a salt, payment parameters, relay bytes a composite
     forwards to a child). MAY be empty; never semantic content, and it never
     chooses the forum. Process parameters that can steer the outcome (which
     system decides, court choice, juror count) SHOULD be fixed by the
     adjudication definition or the Adjudicator's deployment configuration, and
     Adjudicators SHOULD validate activation `data` against them rather than let
     the activator choose. This deliberately departs from the Kleros arbitration
     standard's `extraData` pattern, where the caller picked the court: because
     registration and activation are separate acts and the activator may be
     adverse, whoever activates first would otherwise capture the forum, the
     same attack as evidence capture, applied to process parameters.
   - **Authorization is governed by the adjudication definition and is
     enforceable, because definitions are on-chain attestations the contract can
     read.** Where the definition names the adjudication's parties, the
     Adjudicator SHOULD restrict `activate` to exactly those parties, each
     acting unilaterally, since an adverse counterparty's cooperation cannot be
     required; registering an adjudication confers no activation rights. Open
     activation by anyone is legitimate only where the definition explicitly
     opts into it (e.g., oracle-style questions). Only where the definition
     names no parties and does not opt into open activation SHOULD
     implementations fall back to restricting activation to the registrant.
   - **Timing is governed the same way.** The definition MAY carry an
     `activationNotBefore` timestamp, the earliest time at which the
     adjudication may be activated; where it does, the Adjudicator SHOULD revert
     `activate` before that time. A zero or absent value means activation is
     permitted from registration. This expresses activating conditions that are
     themselves timed: a delivery cannot be in breach before its deadline, a
     market cannot settle before it closes. It matters because the lifecycle is
     forward-only: a premature activation cannot be dismissed, only resolved,
     consuming the adjudication.
   - The function is `payable`; this is typically where adjudication fees are
     due. Amounts, assets, and refund behavior are **implementation-defined**;
     the Adjudicator MUST revert if its payment requirements are not met.
   - On success the Adjudicator MUST emit `AdjudicationActivated` and status
     MUST become `Active`. Because the activator controls activation-time
     evidence, implementations SHOULD give the adjudication's parties an
     opportunity to link evidence after activation.

3. **`status(uint256 adjudicationId) returns (Status)`** where `Status` is the
   enum **`None / Registered / Active / Resolved`**. The function MUST be `view`
   (callable via `staticcall`); pull-based reading depends on it.
   - `None`: no adjudication exists under this id. `None` marks the absence of
     an adjudication, not a state that an adjudication can be in. It MUST be the
     enum's zero value, because an unwritten storage slot reads as zero, and
     zero therefore has to mean "does not exist".
   - `Registered`: the adjudication exists, is referenceable, and is permanent,
     but binds only parties who committed to this `adjudicationId` out of band;
     no adjudication work is happening.
   - `Active`: adjudication is underway with no final resolution yet. All
     intermediate conditions (evidence gathering, deliberation, internal rounds,
     awaiting escalation inside a composite) map to `Active`.
   - `Resolved`: a final resolution exists. Status MUST NOT change once
     `Resolved`. An Adjudicator that adapts another system MUST NOT report
     `Resolved` before that system's outcome is final under its own rules
     (internal appeal rounds, optimistic challenge windows, settlement
     finality): this ERC provides no way to correct an irrevocable resolution
     that the other system later overturns.

4. **`resolution(uint256 adjudicationId) returns (bytes32)`**: the EAS
   attestation UID of the **resolution**. The function MUST be `view` (callable
   via `staticcall`). MUST return the UID once status is `Resolved` and MUST
   revert or return zero before that. Once `Resolved`, the value returned MUST
   equal the `resolutionUID` emitted in the adjudication's `Adjudicated` event
   and MUST NOT ever change. The ERC does not constrain the resolution's
   content; its schema is referenced by the adjudication definition.

5. **[ERC-165](./eip-165.md).** Every Adjudicator MUST implement
   `supportsInterface` and MUST answer `true` for this interface's ID. This is
   REQUIRED (not optional): it is the discovery mechanism for the interface
   itself and for every future extension.

### Events (normative)

- **`AdjudicationRegistered(uint256 indexed adjudicationId, address indexed registrant, bytes32 definitionUID)`**:
  MUST be emitted on registration.
- **`AdjudicationActivated(uint256 indexed adjudicationId, address indexed activator)`**:
  MUST be emitted on activation.
- **`EvidenceLinked(uint256 indexed adjudicationId, bytes32 evidenceUID, address indexed submitter)`**:
  MUST be emitted for each evidence UID linked at activation, and for any
  evidence linked later if the implementation allows it.
- **`Adjudicated(uint256 indexed adjudicationId, bytes32 resolutionUID)`**: MUST
  be emitted exactly once, when the adjudication becomes `Resolved`.

### Lifecycle

`Registered → Active → Resolved`, forward-only, each transition once and
irreversibly. Registration creates the adjudication; before it, `status` reports
`None`.

**No state may be skipped**: an adjudication MUST pass through `Registered` and
`Active` before it can be `Resolved`. A single transaction MAY perform several
transitions back-to-back (e.g., register-and-activate, or a composite resolving
on activation where no response window applies), but each transition MUST occur
and MUST emit its event.

For adjudications whose definition names parties, the Adjudicator SHOULD NOT
resolve in the same transaction that activates: an atomic activate-and-resolve
leaves the counterparty zero time to respond, voiding the post-activation
evidence opportunity.

This does not restrict multi-transition transactions where no adverse party
exists by construction: a test double that resolves on activation, or a
passthrough whose response window elapsed before activation, remain conforming.

Nothing else is standardized. In particular this ERC defines **no** appeal,
escalation, or challenge entry point, and no state to represent them: an
Adjudicator with internal appeal rounds, or a composite awaiting escalation,
simply remains `Active` until its outcome is final.

### Resolution delivery: pull only

The Adjudicator MUST NOT be required to call into any consumer. Consumers
observe the `Adjudicated` event and read `resolution(adjudicationId)`.
Adjudication can therefore never be blocked by a failing consumer, and no
consumer-side interface exists to standardize or to deviate from.

### EAS as the semantic layer

Three kinds of attestations, whose schemas are registered in an EAS schema
registry:

- **Adjudication definition**: the question, the outcome space and its meaning,
  and process parameters (e.g., evidence windows, escalation conditions, who may
  activate and from when, via `activationNotBefore`). It carries or references
  the **baseline**: whatever the adjudication will be decided against, committed
  before the activating condition occurs, which is what makes it tamper-proof
  once a decision is needed. Created before or at registration.
- **Evidence**: material brought to argue an activated adjudication, as content
  or content pointers following the evidence schema. Linked at activation; later
  linking is implementation-defined but MUST emit `EvidenceLinked`. **The
  normative link between evidence and an adjudication is the on-chain
  `EvidenceLinked` event.** An evidence attestation SHOULD additionally
  reference the definition via EAS `refUID` for off-chain traceability, but
  `refUID` alone is not sufficient to bind evidence to an adjudication: one
  definition can serve many adjudications (template adjudicators), so only the
  event is unambiguous.
- **Resolution**: the outcome, referencing the definition it answers.

**Attestation integrity (normative).** The lifecycle's finality guarantee is
only as strong as the content behind the UIDs. The conformance subject of every
rule below is the Adjudicator; each required property is readable on-chain from
EAS, so each check is a view call.

- At registration, the Adjudicator MUST verify that the definition attestation
  exists on-chain on the Adjudicator's own chain, is not revocable, and has an
  `expirationTime` of zero, and MUST revert otherwise. A template Adjudicator
  MUST ensure the same properties for its fixed definition; verifying once at
  deployment suffices, since the properties are immutable. On-chain residency on
  the Adjudicator's chain is also what makes definition-governed rules (such as
  activation authorization) enforceable rather than advisory.
- An adjudication MUST NOT enter `Resolved` unless its resolution attestation
  exists on-chain on the Adjudicator's own chain, is not revocable, and has an
  `expirationTime` of zero. The Adjudicator MUST verify these properties before
  entering `Resolved`.
- Why these properties: a revocable resolution leaves consumers holding an
  immutable pointer to a repudiated statement, and an expiring attestation is a
  scheduled revocation. Under EAS validity semantics a resolution that expires
  stops being valid at a fixed time, and a definition that expires while the
  adjudication is still `Registered` removes the baseline at the moment a
  decision needs it.
- Definition authors SHOULD keep definitions compact, embedding content hashes
  for bulky baseline material rather than mutable references (such as URLs), so
  that substitution or withholding of the baseline is detectable. This guidance
  binds no contract and is deliberately a SHOULD.

### Composition (informative; normative content in the companion ERC)

Because composites implement this same interface, a consumer cannot and need not
distinguish a single system from an escalation chain (e.g., GenLayer → Kleros,
or ending in an off-chain ADR body behind an adapter) or an X-of-Y panel (2-of-3
across GenLayer, Kleros, UMA; 11 off-chain agents by majority). Internally a
composite registers and activates child adjudications on other Adjudicators
through the same standard surface; a "round" is nothing more than a child
adjudication.

The introspection interface for walking a composite's children, the delegation
event, escalation entry points, and their funding patterns are specified in the
companion Composition & Factories ERC, together with factories that instantiate
common composites pre-wired.

### Chain deployment (informative)

The interface is chain-agnostic and expected to be deployed on many EVM chains.
Where an Adjudicator adapts a system deployed elsewhere (another chain, an L2,
or off-chain infrastructure), cross-chain and off-chain mechanics (messages
carrying value, proxy pairs, relayer models) are entirely an adapter concern,
outside this standard.

## Rationale

- **A new interface from zero.** This design derives from no existing
  arbitration interface: it carries over neither a two-sided interface nor
  callback delivery. Existing systems connect through adapters, which they need
  anyway to normalize their differing models.
- **Standardize state and pointers, not behavior.** An interface that
  over-specifies semantics produces implementations that deviate from it.
  Everything semantic lives in EAS attestations, which can evolve per use case
  without redeploying or re-standardizing.
- **The activating condition.** What makes a decision necessary differs by
  adjudication: a counterparty contests a delivery, a deadline passes, a market
  closes, an insured event fires. The interface names none of them; it calls
  that moment the *activating condition* and leaves the definition to say what
  the condition is and who may declare that it occurred. Two things follow.
  Activation may be adversarial or routine, which is why the guidance on
  authorization and response windows turns on whether the definition names
  parties rather than assuming it does. And activation may be certain rather
  than conditional: a contested agreement may never activate, a question
  registered to be answered always will.
- **Registration precedes activation.** Fixing the forum and the question before
  the activating condition occurs is the purpose of the split, and it serves two
  situations. Where activation is conditional, registering ahead of any dispute
  establishes one canonical `adjudicationId`, eliminating the problem of
  competing registrations for agreements with no on-chain deal contract, and
  records the commitment at near-zero cost, while adjudication fees are not due
  until a decision is actually needed. Where activation is certain, the same
  split fixes the question and its outcome space before anyone takes a position
  on it, so whoever registered the adjudication cannot restate it once the
  stakes are known. The naming is deliberately symmetric:
  `register → Registered`, `activate → Active`, resolve → `Resolved`; each state
  is the past participle of the verb that reaches it.
- **Exactly three states.** Any richer enum either enumerates mechanisms
  (appeal, escalation), forming a closed set that new systems would not fit, or
  duplicates information already available from the resolution itself. The
  `Status` enum carries a fourth member, `None`, for a storage reason rather
  than a lifecycle one. An unwritten storage slot reads as zero. If zero meant
  `Registered`, an id that was never registered would report the same status as
  one that was, so zero has to mean "does not exist".
- **Pull-only delivery.** Callback delivery lets a reverting consumer block
  resolution permanently, a known failure mode of callback-based arbitration,
  and requires standardizing a second interface. Pull delivery has neither
  problem.
- **`payable` registration and activation with implementation-defined fee
  semantics.** Some systems (Kleros) require payment in the same transaction
  that starts their underlying adjudication; for an adapter, that transaction is
  activation. The payment *channel* must therefore be standard or adapters are
  impossible; fee *semantics* (amounts, assets, refunds, who pays) differ so
  much across systems that no single standard for them would hold. Multi-stage
  funding follows the same logic: activation funds the first round; later rounds
  are funded through the composite's own entry points when, and only when, they
  are needed.
- **ERC-165 required.** Ten lines of boilerplate give unambiguous discovery
  today, and automatic detection of every future extension.
- **Deferred extensions, explicitly not specified now.** A `Provisional` status
  with a standard `challenge()` entry point (a fifth status signaling that a
  provisional resolution exists and an action window is open; deferred because
  challenge actions are inseparable from fees, which are out of scope). A
  `cost()` fee-query view (a standard way to ask what `registerAdjudication` or
  `activate` requires; callers learn costs from the adjudication definition or
  off-chain). Capability introspection (enumerating available actions per
  adjudication). A consumer callback (optional push notification, best-effort,
  never blocking resolution).

### Prior Art

The Kleros arbitration proposal (numbered 792 in the Kleros ecosystem, never
merged into the official proposal repositories) defines a two-sided
arbitrator/arbitrable pair with callback-based delivery. This ERC constrains
only the Adjudicator side, imposes no interface on Adjudicables, and delivers
resolutions by pull, so a reverting consumer cannot block resolution.

Its evidence companion (numbered 1497) specifies evidence and meta-evidence
document formats, and saw implementations deviate significantly from the
specification. This ERC keeps document semantics out of the interface: the
meaning of the adjudication, the evidence, and the meaning of the outcome are
EAS attestation pointers, which evolve per use case without touching deployed
contracts.

[ERC-8033](./eip-8033.md) standardizes multi-agent council oracles for
information queries, one resolution mechanism. This ERC standardizes a
mechanism-agnostic surface, and such a system connects to it through an adapter.

## Security Considerations

Resolution finality is the primary guarantee: once `Resolved`, status and
resolution UID are immutable, and because the Adjudicator must verify, before
entering `Resolved`, that the resolution attestation is on-chain, non-revocable,
and non-expiring (see *Attestation integrity*), the content behind the UID is
immutable too, so consumers can safely act on them.

Trust in *what* the resolution says is exactly trust in the chosen Adjudicator;
the standard makes the choice explicit and swappable but does not reduce it.

Registration is permissionless in principle, so duplicate or orphan
registrations can exist. They bind nobody, because an adjudication only binds
whoever committed to its specific `adjudicationId` in advance: an Adjudicable
that registered the adjudication itself and stored the returned id, or parties
who registered during negotiation and referenced the id in their signed
agreement.

Since that commitment happens while the participants still cooperate, exactly
one canonical adjudication exists per agreement, and an adjudication registered
by anyone else is an orphan that no consumer reads.

**Activation as an attack surface.** A malicious activator cannot forge an
outcome; resolutions come from the Adjudicator. The remaining vectors are
*baseless activation*, *evidence capture* (an unauthorized or front-running
activator supplying misleading activation-time evidence and hoping for a default
outcome before the real parties respond), and *parameter capture* (the first
activator's `data` choosing outcome-steering process parameters such as the
forum).

Parameter capture is closed by the interface rule above: such parameters are
fixed by the definition or deployment configuration, and activation `data` is
validated against them, never trusted to choose.

Evidence capture is addressed by the normative guidance above: parties should be
able to link evidence after activation, and `AdjudicationActivated` is an
indexed event, so parties, or their monitoring agents, are expected to watch
their registered adjudications rather than assume that no notification means no
activation.

Monitoring defends only when `Active` lasts long enough to respond. The
lifecycle permits atomic activate-and-resolve; the Lifecycle section discourages
it for party-named adjudications but cannot forbid it. A minimum `Active`
duration is therefore an adjudicator property, exactly like error rate and
liveness, and parties should verify it before committing to an `adjudicationId`.

**Activation is a unilateral right to impose adjudication, and its duration, on
the counterparty.**

The activation fee bounds only the attacker's cost, not the defender's loss: it
compensates the Adjudicator, not the victim, while the victim bears the cost of
capital locked for the full adjudication duration, repeatable every settlement
cycle.

Nor is a baseless activation guaranteed to fail: every adjudication system has
an error rate, so against a large enough stake a cheap activation on a fallible
Adjudicator can carry positive expected value for the attacker.

Authorization narrows the attacker set (where the definition names the parties,
only a counterparty can activate), but a counterparty is precisely who holds
this option. An `activationNotBefore` timestamp in the definition narrows the
window instead: activation before that timestamp reverts, but from that time on
the option is intact.

Sizing value-at-stake against the chosen Adjudicator's fees, error rate, and
adjudication duration is therefore a consumer-side decision, exactly like the
liveness assumptions below.

**Liveness: withheld outcomes.** The guarantees above cover forged outcomes;
they do not cover outcomes that never arrive. The lifecycle is forward-only with
no terminal failure state, no timeout, and pull-only delivery, so nothing in
this standard ever forces an Adjudicator out of `Active`.

Concretely: an adjudicator whose resolving key is lost, or whose operator
withholds the resolution to extort the parties, leaves every dependent
adjudication `Active` forever; parties who settle by agreement after activation
have no exit even by unanimous consent, since only the Adjudicator can reach
`Resolved`; and a composite's liveness is the minimum of its children's, not the
majority's, so an X-of-Y panel with one permanently stalled child it still needs
is stalled itself.

**Resolution liveness is therefore part of the trust placed in the chosen
Adjudicator**, exactly like resolution honesty, and choosing an Adjudicator
means accepting its liveness assumptions.

Accordingly: Adjudicables should not make irreversible commitments whose only
exit is `Resolved`, and should implement a fallback for an Adjudicator that
never resolves: a deadline after which a default path applies, an alternate
resolution route, or a mutual-release mechanism the parties can invoke by
consent.

Adjudicator implementations and composites are free to add their own liveness
mechanisms (operator timeouts, panels that tolerate stalled children); those are
implementation concerns outside this standard.

**Resolution timing.** The Adjudicator's operator knows the outcome before the
chain does, and choosing the publication moment is an extractable advantage
wherever markets consume resolutions (prediction markets, insurance). The
standard accepts this discretion; it is part of the trust placed in the chosen
Adjudicator, alongside honesty, liveness, and minimum `Active` duration.

Adapters to other chains or off-chain systems add their own trust assumptions
(authorized submitter accounts, bridges); these belong to the adapter's
documentation, not this standard.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
