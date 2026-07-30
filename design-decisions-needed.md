# Design Decisions Needed

Running to-do list of open design decisions for the Composable Adjudication ERCs. The ERC drafts stay clean and self-contained; anything still undecided lives here, with enough description to pick it up cold. When a decision is made: apply it to the drafts, record it in `AGENTS.md`, and delete it from this file.

## Core ERC

### 1. EAS schema contents
The three schemas need concrete field lists registered in the EAS schema registry: **adjudication definition** (question, outcome space and meaning, baseline references, parties, activation authorization, process-parameter echoes), **evidence** (content or pointer + hash, submitter context, possibly a type tag), and **resolution** (outcome, reference to the definition it answers). **Priority raised (2026-07-30):** definitions are now mandatory on-chain and Adjudicators enforce activation authorization by decoding them, so at minimum the party-list/authorization field encoding must be normative. Until it is, enforcement is per-implementation decoding.

### 2. Multi-chain attestation references and hosting chain
Partially settled (2026-07-30): **definition attestations MUST live on the Adjudicator's own chain** (on-chain readability for enforcement). Remaining: hosting and referencing conventions for evidence (read by adjudication systems off-chain, so flexibility may be acceptable); whether resolution attestations must also be same-chain; and the general referencing convention where cross-chain pointers are unavoidable (a bare 32-byte UID does not name a chain).

### 3. ERC-165 interface IDs
Mechanical: compute and freeze the interface IDs once the function signatures stop moving.

### 4. Token fee channel
`payable` channels only the native asset, yet several adjudication systems price fees in ERC-20 tokens (stable across the potentially long registration→activation gap, where a fee fixed in native value drifts). An adapter fronting a token-priced system has no standard way to receive its fee: `data` is barred from carrying semantic content, and ad-hoc approve-then-pull conventions would diverge per adapter, recreating the fragmentation this ERC exists to remove. Sharpest in composites, where an escalation router must fund children whose fees may be denominated in different assets. Decide: standardize a token payment path (e.g., a normatively blessed approve-and-pull pattern in which the Adjudicator announces its fee token) or keep the channel native-only and push token handling into adapters.

## Composition & Factories ERC

### 5. Escalation window source
The outline currently says the router's escalation window length comes from the EAS adjudication definition. That conflicts with two settled principles: enforceable parameters need an on-chain source readable by the composite, and the definition describes the case, not the composite. Likely resolution: the window comes from the composite's instantiation configuration (or activation `data`), with the definition merely echoing it for readers. The outline must be corrected once decided.

### 6. X-of-Y agreement semantics
What "X children agree" means over EAS-defined resolutions. Likely equality of a designated outcome field in the resolution schema, but the aggregation rule and its reference (instantiation config vs. definition) need specification.

### 7. Escalation funding patterns (parked as future work)
Pay-as-you-go vs. pre-funded budget. Already moved to the outline's Future Work section; listed here only so the parked status is visible in one place.

---

Resolved and removed (recorded in `AGENTS.md`): `registerAdjudication` parameter shape (settled as-is); register-and-activate single-transaction convenience (removed from the spec as unneeded); zero `definitionUID` (allowed: Adjudicator-supplied definition; non-templates revert).
