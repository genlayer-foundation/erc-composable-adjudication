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

This ERC defines a small interface for on-chain **adjudication**. Adjudication
is the submission of a question for a decision, and the use of the resolution
that comes back.

A contract that implements the interface is an **Adjudicator**. A contract that
registers adjudications and acts on their resolutions is an **Adjudicable**.

An adjudication has three states. An account registers the adjudication before a
decision is necessary. An authorized account activates the adjudication when its
activating condition occurs. The Adjudicator resolves the adjudication when the
outcome is final. Consumers read resolutions. The Adjudicator does not send
them.

Ethereum Attestation Service (EAS) attestation pointers carry all of the
semantic content. These semantics are the question, the evidence, and the
meaning of the outcome. The interface holds none of this content on-chain.

The interface makes no assumption about how an Adjudicator reaches a decision.
Therefore a composite of Adjudicators can also implement the interface. This
property makes adjudication composable. A companion ERC specifies the
composition interfaces and a set of ready-made composites.

## Motivation

The adjudication systems in production today have different interfaces. These
systems include GenLayer, Kleros, UMA, and single off-chain agents that operate
a wallet. There are three consequences:

1. **Integration cost.** A protocol that needs both deterministic and subjective
   resolutions must build a different integration for each system.
2. **No composition.** There is no standard way to combine systems. An
   escalation chain and a panel of independent systems are both impossible.
3. **Lock-in.** A protocol that selects an adjudication system at integration
   time cannot easily change that system later.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Roles

- **Adjudicator**: a contract that implements this interface. It accepts
  adjudications and resolves them later. An Adjudicator can be a native system.
  It can also be an adapter in front of another system such as GenLayer, Kleros,
  or UMA. It can also be a composite of other Adjudicators.
- **Adjudicable**: a contract or an account that registers an adjudication and
  reads its resolution. This ERC puts **no interface requirement** on
  Adjudicables.

### Core interface (normative, in words)

An Adjudicator MUST supply the five items that follow.

1. **`registerAdjudication(bytes32 definitionUID, bytes data) payable returns (uint256 adjudicationId)`**
   This function records a new adjudication. It MUST NOT start adjudication
   work. It sets the forum and the question before the activating condition
   occurs. A caller can therefore call it at commitment time, before a decision
   is necessary.
   - `definitionUID` identifies the **adjudication definition**. It is the EAS
     attestation UID of that definition. The definition states what the
     adjudication decides, the possible outcomes and their meaning, and the
     process parameters. The definition also carries or references the full
     baseline. The baseline is the agreement, the question, or the state of the
     world that the adjudication is decided against.
   - Evidence, as an interface concept, has one function only: to argue an
     activated adjudication. Registration therefore needs no such parameter.
   - `definitionUID` MAY be zero. A zero value means that the Adjudicator
     supplies the definition. Such an Adjudicator is a template Adjudicator, and
     its definition is fixed at deployment. An implementation that requires a
     definition from the caller MUST revert on a zero value.
   - The opposite rule also applies. A template Adjudicator MUST revert on a
     nonzero `definitionUID` that is different from the UID of its fixed
     definition. A caller can therefore never register an adjudication under
     definition X while definition Y governs that adjudication.
   - The Adjudicator MUST put the definition UID that governs the adjudication
     into the `AdjudicationRegistered` event. A template Adjudicator MUST emit
     the UID of its fixed definition, and MUST NOT emit zero. Indexers therefore
     see one uniform stream.
   - `data` holds extra call parameters. Their content is
     **implementation-defined**. `data` MAY be empty. Semantic content MUST NOT
     travel here. Such content stays behind EAS pointers.
   - The function is `payable`. Whether the Adjudicator requires payment is
     **implementation-defined**. Registration SHOULD be cheap or free.
     Adjudication fees belong to activation.
   - `adjudicationId` MUST be unique in the Adjudicator. The Adjudicator MUST
     NOT reuse an id. An id MUST depend only on the inputs of its own
     registration. An id MUST NOT depend on other registrations, and MUST NOT
     depend on shared mutable state such as a counter. The registration of one
     party therefore cannot change the id that another registration receives.
     Sequential assignment does not conform.
   - Derivation of the id from the inputs of the registration is RECOMMENDED. An
     example is a hash over the registrant, the `definitionUID`, and a salt from
     the caller. This method lets an account calculate the id in advance and
     reference it off-chain before the registration occurs. The link between the
     id and the registrant also prevents occupation of that id by a different
     account.
   - Under this method the input tuple MUST be unique for each registration. The
     salt, or an equivalent input, gives this property. If a registration
     derives an id that already exists, the Adjudicator MUST revert. It MUST NOT
     reuse or overwrite that id.
   - Registration MUST NOT fail only because the same `definitionUID` was
     registered before. One definition serves many adjudications, and the salt
     keeps them distinct.
   - On success the Adjudicator MUST emit `AdjudicationRegistered`. The status
     of the adjudication MUST become `Registered`.

2. **`activate(uint256 adjudicationId, bytes32[] evidenceUIDs, bytes data) payable`**
   This function begins work on a registered adjudication. A caller calls it at
   the moment when the activating condition occurs and a decision becomes
   necessary.
   - The Adjudicator MUST revert if the status is not `Registered`.
   - `evidenceUIDs` holds the EAS attestation UIDs of the evidence that the
     activator brings. The Adjudicator MUST link this evidence in the same
     transaction as the activation. The Adjudicator MUST emit `EvidenceLinked`
     for each UID.
   - `evidenceUIDs` MAY be empty. Some adjudications need nothing more than the
     baseline.
   - This ERC puts no validity requirement on an evidence UID. A UID is not
     required to reference an attestation that exists. Each implementation
     decides whether it validates evidence UIDs, and how. Different answers here
     still conform.
   - `data` holds extra call parameters, in the same way as the `data` parameter
     of registration. Examples are a salt, payment data, and relay bytes that a
     composite sends to a child. `data` MAY be empty. It MUST NOT hold semantic
     content. It MUST NOT select the forum.
   - Some process parameters can steer the outcome. Examples are the system that
     decides, the choice of court, and the number of jurors. The adjudication
     definition or the deployment configuration of the Adjudicator SHOULD fix
     these parameters. An Adjudicator SHOULD validate the activation `data`
     against them. An Adjudicator SHOULD NOT let the activator select them.
   - This rule is different from the `extraData` pattern of the Kleros
     arbitration proposal, where the caller selected the court. Registration and
     activation are separate acts, and the activator can be an adverse party.
     Without this rule, the first account to activate captures the forum. This
     is the same attack as evidence capture, one layer higher.
   - **The adjudication definition governs authorization.** This rule is
     enforceable, because a definition is an on-chain attestation that the
     contract can read.
   - If the definition names the parties to the adjudication, the Adjudicator
     SHOULD permit `activate` to those parties only. Each party acts alone,
     because an adverse counterparty cannot be required to cooperate.
     Registration of an adjudication gives no right to activate it.
   - Activation by any account is correct only if the definition permits it.
     Oracle-style questions are the usual example.
   - If the definition names no parties, and does not permit activation by any
     account, implementations SHOULD permit `activate` to the registrant only.
   - **The definition governs timing in the same way.** The definition MAY hold
     an `activationNotBefore` timestamp. This timestamp is the earliest time at
     which an account can activate the adjudication. If the definition holds
     this value, the Adjudicator SHOULD revert `activate` before that time. A
     zero value, or no value, permits activation from registration.
   - `activationNotBefore` expresses an activating condition that has a time
     component. A delivery cannot be in breach before its deadline. A market
     cannot settle before it closes. This matters because the lifecycle moves
     forward only. Nobody can dismiss a premature activation. The Adjudicator
     can only resolve it, and the adjudication is then consumed.
   - The function is `payable`. Adjudication fees are usually due here. The
     amounts, the assets, and the refund behavior are
     **implementation-defined**. The Adjudicator MUST revert if the caller does
     not meet its payment requirements.
   - On success the Adjudicator MUST emit `AdjudicationActivated`. The status
     MUST become `Active`.
   - The activator controls the evidence that is present at activation.
     Implementations SHOULD therefore give the parties to the adjudication an
     opportunity to link evidence after activation.

3. **`status(uint256 adjudicationId) returns (Status)`** where `Status` is the
   enum **`None / Registered / Active / Resolved`**. The function MUST be a
   `view` function, so that a caller can use `staticcall`. A consumer depends on
   this property when it reads the status.
   - `None`: no adjudication exists under this id. This value is the absence of
     an adjudication, not a state of one. `None` MUST be the zero value of the
     enum. An unwritten storage slot reads as zero, so zero must mean "does not
     exist".
   - `Registered`: the adjudication is on record. It is permanent, and other
     contracts can reference it. It binds only the accounts that committed to
     this `adjudicationId` off-chain. The adjudication is dormant, and no
     adjudication work occurs.
   - `Active`: adjudication is in progress, and no final resolution exists yet.
     All intermediate conditions map to `Active`. Examples are the collection of
     evidence, deliberation, internal rounds, and a wait for escalation inside a
     composite.
   - `Resolved`: a final resolution exists. The status MUST NOT change after it
     becomes `Resolved`. An Adjudicator that represents another system MUST NOT
     report `Resolved` before the outcome of that system is final under the
     rules of that system. Examples of such rules are internal appeal rounds,
     optimistic challenge windows, and settlement finality. This ERC gives no
     way to correct an irrevocable resolution that the other system overturns
     later.

4. **`resolution(uint256 adjudicationId) returns (bytes32)`** returns the EAS
   attestation UID of the **resolution**. The function MUST be a `view`
   function, so that a caller can use `staticcall`. The function MUST return the
   UID after the status becomes `Resolved`. Before that point it MUST revert or
   return zero. After the status becomes `Resolved`, the returned value MUST be
   equal to the `resolutionUID` in the `Adjudicated` event of the adjudication,
   and MUST NOT change.
   - This ERC puts no constraint on the content of the resolution. The
     adjudication definition references the schema of that content.

5. **[ERC-165](./eip-165.md).** Every Adjudicator MUST implement
   `supportsInterface`. It MUST return `true` for the ID of this interface. This
   is REQUIRED, not optional. ERC-165 is the discovery mechanism for the
   interface, and the rail for all future extensions.

### Events (normative)

- **`AdjudicationRegistered(uint256 indexed adjudicationId, address indexed registrant, bytes32 definitionUID)`**:
  the Adjudicator MUST emit this event when it registers an adjudication.
- **`AdjudicationActivated(uint256 indexed adjudicationId, address indexed activator)`**:
  the Adjudicator MUST emit this event when it activates an adjudication.
- **`EvidenceLinked(uint256 indexed adjudicationId, bytes32 evidenceUID, address indexed submitter)`**:
  the Adjudicator MUST emit this event for each evidence UID that it links at
  activation. It MUST also emit this event for evidence that it links later, if
  the implementation permits later evidence.
- **`Adjudicated(uint256 indexed adjudicationId, bytes32 resolutionUID)`**: the
  Adjudicator MUST emit this event one time only, when the adjudication becomes
  `Resolved`.

### Lifecycle

The lifecycle is `Registered → Active → Resolved`. It moves forward only. Each
transition occurs one time, and no transition can be reversed. Registration
creates the adjudication. Before registration, `status` returns `None`.

**An adjudication MUST NOT skip a state.** It MUST pass through `Registered` and
`Active` before it can become `Resolved`. One transaction MAY make more than one
transition. Two examples are a transaction that registers and activates, and a
composite that resolves at activation because no response window applies. Each
transition MUST occur, and the Adjudicator MUST emit the event for each one.

If the definition of an adjudication names the parties, the Adjudicator SHOULD
NOT resolve that adjudication in the transaction that activates it. An
activation and a resolution in one transaction give the counterparty no time to
respond. Such a transaction also removes the opportunity to link evidence after
activation.

This guidance does not restrict a transaction with more than one transition
where no adverse party exists. Two examples still conform. The first is a test
double that resolves at activation. The second is a passthrough whose response
window ended before the activation.

This ERC specifies nothing more about the lifecycle. It defines **no** entry
point for an appeal, an escalation, or a challenge. It defines no state for
them. An Adjudicator with internal appeal rounds stays `Active` until its
outcome is final. A composite that waits for an escalation does the same.

### Resolution delivery: pull only

No rule in this ERC requires an Adjudicator to call a consumer. A consumer
monitors the `Adjudicated` event and then reads `resolution(adjudicationId)`. A
consumer that reverts therefore cannot stop an adjudication. There is also no
consumer interface for this ERC to specify, and none for an implementation to
deviate from.

### EAS as the semantic layer

There are three kinds of attestation. An EAS schema registry holds the schema of
each kind.

- **Adjudication definition**: the question, the outcome space and the meaning
  of each outcome, and the process parameters. Examples of a process parameter
  are an evidence window, an escalation condition, and the accounts that can
  activate the adjudication. The `activationNotBefore` field gives the earliest
  time to activate. An account creates the definition before registration, or at
  registration.
- **The baseline.** The definition also carries or references the **baseline**.
  The baseline is the material that the adjudication is decided against, and an
  account commits it before the activating condition occurs. The baseline is
  therefore tamper-proof at the time when a decision becomes necessary.
- **Evidence**: the material that an account brings to argue an activated
  adjudication. It is content, or a pointer to content, in the evidence schema.
  An account links evidence at activation. Whether an implementation permits a
  link later is implementation-defined, but it MUST emit `EvidenceLinked` for
  each such link.
- **The on-chain `EvidenceLinked` event is the normative link between a piece of
  evidence and an adjudication.** An evidence attestation SHOULD also reference
  the definition through the EAS `refUID` field, for off-chain traceability. A
  `refUID` value alone does not bind evidence to an adjudication. One definition
  can serve many adjudications, because of template adjudicators. Only the event
  is unambiguous.
- **Resolution**: the outcome. It references the definition that it answers.

**Attestation integrity (normative).** The finality guarantee of the lifecycle
is only as strong as the content behind the UIDs. The Adjudicator is the
conformance subject of each rule below. EAS holds each required property
on-chain, so each check is a `view` call.

- At registration, the Adjudicator MUST verify three properties of the
  definition attestation. It MUST exist on-chain, on the chain of the
  Adjudicator. It MUST NOT be revocable. Its `expirationTime` MUST be zero. The
  Adjudicator MUST revert if any property fails.
- A template Adjudicator MUST have the same three properties for its fixed
  definition. One check at deployment is sufficient, because the properties
  cannot change.
- On-chain residency on the chain of the Adjudicator also makes the rules that
  the definition governs enforceable. Activation authorization is one such rule.
  Without residency, these rules are advisory only.
- An adjudication MUST NOT become `Resolved` unless its resolution attestation
  has the same three properties. It MUST exist on-chain, on the chain of the
  Adjudicator. It MUST NOT be revocable. Its `expirationTime` MUST be zero. The
  Adjudicator MUST verify these properties before it enters `Resolved`.
- These properties are necessary for two reasons. A revocable resolution leaves
  a consumer with an immutable pointer to a repudiated statement. An attestation
  that expires is a revocation on a timer. A resolution that expires repudiates
  itself on a schedule, under the validity rules of EAS. A definition that
  expires during a long dormancy in `Registered` removes the baseline at the
  exact moment when a decision needs it.
- Definition authors SHOULD keep a definition compact. For bulky baseline
  material, a definition SHOULD hold a content hash instead of a mutable
  reference such as a URL. A hash lets an account detect a substitution of the
  baseline. It also lets an account detect a baseline that somebody keeps back.
  This guidance binds no contract, and it is a SHOULD by design.

### Composition (informative, normative content in the companion ERC)

A composite implements this same interface. A consumer therefore cannot see the
difference between one system and a composite, and it does not need to see it.

An escalation chain is one kind of composite. An example is GenLayer, then
Kleros, then an off-chain ADR body behind an adapter. An X-of-Y panel is another
kind. Two examples are 2-of-3 across GenLayer, Kleros, and UMA, and 11 off-chain
agents that decide by majority.

Inside a composite, the composite registers and activates child adjudications on
other Adjudicators. It uses this same interface for that purpose. A "round" is
only a child adjudication.

The companion Composition & Factories ERC specifies the remainder. It specifies
the introspection interface for the children of a composite, the delegation
event, the escalation entry points, and the funding patterns for them. It also
specifies factories that create common composites with the connections already
made.

### Chain deployment (informative)

The interface does not depend on one chain. Deployment on many EVM chains is
expected. An Adjudicator can represent a system that lives elsewhere, on another
chain, on an L2, or off-chain. The cross-chain and off-chain mechanics are then
a concern of the adapter only. Examples are messages that carry value, proxy
pairs, and relayer models. This ERC does not see them.

## Rationale

- **A new interface from zero.** This design comes from no arbitration interface
  that exists today. It does not use a two-sided interface, and it does not use
  callback delivery. An adapter connects each system that is already in
  production to this interface. Each such system needs an adapter anyway, to
  normalize its own model.
- **Specify state and pointers, not behavior.** An interface that specifies too
  much semantic content invites deviation. All semantic content is in EAS
  attestations instead. These attestations can change for each use case. Such a
  change needs no new deployment and no new standard.
- **The activating condition.** What makes a decision necessary is different in
  each adjudication. A counterparty contests a delivery. A deadline passes. A
  market closes. An insured event occurs. The interface names none of these
  events.
  - The interface calls that moment the *activating condition*. The definition
    states what the condition is, and which accounts can declare that the
    condition occurred.
  - An activation can be adverse, or it can be routine. The guidance on
    authorization and on response windows therefore depends on whether the
    definition names the parties.
  - An activation can also be certain, not conditional. A contested agreement
    can stay `Registered` forever. A question that an account registers for an
    answer always activates.
- **Registration precedes activation.** The split sets the forum and the
  question before the activating condition occurs. This gives a benefit in two
  different situations.
  - Where activation is conditional, registration before any dispute establishes
    one canonical `adjudicationId` from the start. This removes the problem of
    two adjudications that compete for the same agreement with no on-chain
    contract. The commitment goes on record at almost no cost. The adjudication
    fees wait until a decision becomes necessary.
  - Where activation is certain, the same split fixes the question and its
    outcome space before any account takes a position on it. The account that
    registered the adjudication therefore cannot restate the question after the
    stakes are known.
  - The names are symmetric by design: `register → Registered`,
    `activate → Active`, and `resolve → Resolved`. Each state is the past
    participle of the verb that reaches it.
- **Exactly three states.** A larger enum has one of two faults. It enumerates
  mechanisms such as an appeal or an escalation, and that set is closed, so new
  systems do not fit it. Or it duplicates information that the resolution
  already gives.
  - The `Status` enum has a fourth member, `None`. The reason is storage, not
    the lifecycle. An unwritten slot reads as the zero value of the enum. Zero
    must therefore mean "does not exist". Without this rule, an adjudication
    that does not exist looks the same as a registered adjudication.
- **Pull-only delivery.** With callback delivery, a consumer that reverts can
  block a resolution permanently. This is a known failure of callback-based
  arbitration. Callback delivery also needs a second standard interface. Pull
  delivery has neither problem.
- **`payable` registration and activation, with implementation-defined fee
  semantics.** Some systems need payment in the same transaction that starts
  their own adjudication. Kleros is one example. For an adapter, that
  transaction is the activation. The payment *channel* must therefore be
  standard, or an adapter is impossible.
  - Fee *semantics* are different across systems. The amounts, the assets, the
    refunds, and the payer all vary. A standard for them would be false
    precision.
  - Multi-stage funding follows the same logic. The activation funds the first
    round. The entry points of the composite fund the later rounds, and only
    when those rounds become necessary.
- **ERC-165 required.** Ten lines of boilerplate give unambiguous discovery
  today. They also give automatic detection for every future extension.
- **Deferred extensions, not specified now.** This ERC does not specify the four
  items that follow.
  - A `Provisional` status with a standard `challenge()` entry point. This fifth
    status signals that a provisional resolution exists and that an action
    window is open. It is deferred because a challenge action cannot be
    separated from fees, and fees are out of scope.
  - A `cost()` fee-query view. This is a standard way to ask what
    `registerAdjudication` or `activate` needs. A caller learns the costs from
    the adjudication definition, or off-chain.
  - Capability introspection. This enumerates the available actions for each
    adjudication.
  - A consumer callback. This is an optional notification, on a best-effort
    basis, that never blocks a resolution.

### Prior Art

The Kleros arbitration proposal defines a two-sided arbitrator/arbitrable pair
with callback delivery. It is numbered 792 in the Kleros ecosystem, and it never
merged into the official proposal repositories. This ERC constrains the
Adjudicator side only. It puts no interface requirement on an Adjudicable, and a
consumer reads the resolution. A consumer that reverts therefore cannot block a
resolution.

The evidence companion of that proposal is numbered 1497. It specifies document
formats for evidence and meta-evidence. Implementations of it deviated from the
specification by a large margin. This ERC keeps document semantics out of the
interface. The meaning of the adjudication, the evidence, and the meaning of the
outcome are EAS attestation pointers. They can change for each use case, and a
deployed contract does not change with them.

[ERC-8033](./eip-8033.md) specifies multi-agent council oracles for information
queries. That is one resolution mechanism. This ERC specifies a surface that
does not depend on a mechanism. Such a system connects to that surface through
an adapter.

## Security Considerations

Resolution finality is the most important guarantee. After the status becomes
`Resolved`, the status and the resolution UID are immutable. The Adjudicator
must also verify, before it enters `Resolved`, that the resolution attestation
is on-chain, is not revocable, and does not expire. See *Attestation integrity*
above. The content behind the UID is therefore immutable too, and a consumer can
act on it safely.

Trust in *what* the resolution says is trust in the chosen Adjudicator. This ERC
makes that choice explicit, and it makes the Adjudicator replaceable. It does
not reduce the trust.

Registration is permissionless in principle. Duplicate registrations and orphan
registrations can therefore exist. They bind nobody. An adjudication binds only
an account that committed to its specific `adjudicationId` in advance.

There are two such accounts. The first is an Adjudicable that registered the
adjudication and stored the returned id. The second is a party that registered
the adjudication during a negotiation and referenced the id in the signed
agreement.

That commitment occurs while the participants still cooperate. Exactly one
canonical adjudication therefore exists for each agreement. An adjudication that
a different account registers is an orphan, and no consumer ever reads it.

**Activation as an attack surface.** A malicious activator cannot forge an
outcome, because the resolution comes from the Adjudicator. Three attack vectors
remain.

- *Baseless activation*: an activator starts an adjudication that has no merit.
- *Evidence capture*: an unauthorized activator, or an activator that acts
  before the parties do, links evidence at activation that misleads the
  Adjudicator. The attacker hopes for a default outcome before the real parties
  respond.
- *Parameter capture*: the first activator uses the `data` parameter to select
  process parameters that steer the outcome, such as the forum.

The interface rule above closes parameter capture. The definition or the
deployment configuration fixes such parameters. The Adjudicator validates the
activation `data` against them, and never lets the activator select them.

The normative guidance above addresses evidence capture. The parties should be
able to link evidence after activation. `AdjudicationActivated` is an indexed
event. The parties, or the agents that monitor for them, are expected to watch
their registered adjudications. Silence does not mean safety.

This defense works only while `Active` lasts long enough for a response. The
lifecycle permits an activation and a resolution in one transaction. The
Lifecycle section discourages this for an adjudication that names parties, but
cannot forbid it. A minimum duration for `Active` is therefore a property of the
Adjudicator, in the same way as its error rate and its liveness. The parties
should verify this property before they commit to an `adjudicationId`.

Baseless activation needs an honest statement, not a false comfort. **An
activation is a unilateral right to impose an adjudication, and its duration, on
the counterparty.**

The activation fee bounds the cost to the attacker only. It does not bound the
loss to the defender. The fee pays the Adjudicator, not the victim. The victim
carries the cost of the capital that stays locked for the full duration of the
adjudication. The attacker can repeat the attack in every settlement cycle.

A baseless activation is also not certain to fail. Every adjudication system has
an error rate. Against a large enough stake, a cheap activation on a fallible
Adjudicator can have a positive expected value for the attacker.

Authorization makes the set of attackers smaller. Where the definition names the
parties, only a counterparty can activate. But a counterparty is exactly the
account that holds this option. An `activationNotBefore` timestamp in the
definition makes the window smaller instead. An activation before that time
reverts. From that time forward, the option is intact.

A consumer must therefore compare its value at stake against the fees, the error
rate, and the adjudication duration of the chosen Adjudicator. This is a
consumer-side decision, in the same way as the liveness assumptions below.

**Liveness: withheld outcomes.** The guarantees above cover a forged outcome.
They do not cover an outcome that never arrives. The lifecycle moves forward
only. There is no terminal failure state and no timeout, and delivery is
pull-only. Nothing in this ERC forces an Adjudicator out of `Active`. Three
concrete examples follow.

- An Adjudicator that loses its resolving key leaves every dependent
  adjudication `Active` forever. An operator that withholds a resolution to
  extort the parties has the same effect.
- Parties that settle amicably after an activation have no exit, even by
  unanimous consent. Only the Adjudicator can reach `Resolved`.
- The liveness of a composite is the minimum of the liveness of its children,
  not the majority. An X-of-Y panel with one permanently stalled child that it
  still needs is stalled itself.

**Resolution liveness is therefore part of the trust in the chosen
Adjudicator**, in the same way as resolution honesty. To choose an Adjudicator
is to accept its liveness assumptions.

An Adjudicable should therefore not make an irreversible commitment whose only
exit is `Resolved`. It should implement a fallback for an Adjudicator that never
resolves. Three examples are a deadline after which a default path applies, an
alternate resolution route, and a mutual-release mechanism that the parties
invoke by consent.

An Adjudicator implementation and a composite can add their own liveness
mechanisms. Examples are operator timeouts, and panels that tolerate a stalled
child. These are implementation concerns, outside this ERC.

**Resolution timing.** The operator of the Adjudicator knows the outcome before
the chain knows it. The choice of the moment to publish is therefore an
extractable advantage, wherever a market consumes resolutions. Prediction
markets and insurance are two such markets. This ERC accepts this discretion. It
is part of the trust in the chosen Adjudicator, together with honesty, liveness,
and the minimum duration of `Active`.

An adapter in front of another chain, or in front of an off-chain system, adds
its own trust assumptions. Two examples are authorized submitter accounts and
bridges. These assumptions belong in the documentation of the adapter, not in
this ERC.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
