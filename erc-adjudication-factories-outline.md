# ERC #2: Adjudication Composition & Factories (Idea Outline)

> Companion to the core Composable Adjudication ERC. Idea-level outline, not yet a draft.

## Motivation

The core ERC makes composition *possible* (a composite is just another Adjudicator); this ERC makes it *practical*. Builders should not hand-write escalation routers or panel aggregators: all the glue code exists as audited implementations, and a builder instantiates the setup they need from a factory in one transaction.

## Part 1: Composition interface (the piece the core ERC deliberately omits)

The composite introspection extension, ERC-165-detectable:

- **`children(adjudicationId) → (address childAdjudicator, uint256 childAdjudicationId)[]`**: the ordered list of child adjudications registered for this case. Order is meaningful: for an escalation router it is the round sequence; for a panel it is the member set.
- **Event `AdjudicationDelegated(adjudicationId, childAdjudicator, childAdjudicationId)`**: emitted when a composite **registers** a child adjudication (whether or not it activates the child in the same transaction). The child's own `AdjudicationRegistered` / `AdjudicationActivated` / `Adjudicated` events carry its lifecycle from there.

With only this, any UI, indexer, or agent can walk the tree from a top-level Adjudicator down to whichever system is currently deciding: escalation and round visibility without any status-enum complexity in the core. Leaves simply answer `false` to `supportsInterface` for this extension's ID.

## Part 2: Composite catalog (v0.1)

Each composite implements the core Adjudicator interface plus the composition extension:

1. **Single-adjudicator wrapper**: binary resolution on one system (GenLayer only; Kleros only). Mostly a pass-through; exists so every setup is factory-instantiated the same way.
2. **Escalation router** (GenLayer → Kleros): activating the router registers and activates the GenLayer child; when that child resolves, an escalation window starts (window length from the EAS adjudication definition). If unescalated, the router adopts the child resolution and emits `Adjudicated`. If escalated, the router registers and activates the Kleros child and stays `Active`. Escalation is requested through the router's own `payable` entry point (e.g., `escalate(adjudicationId)`); how escalation is funded is future work.
3. **X-of-Y panel**: registers and activates child adjudications on Y Adjudicators at once (2-of-3 across GenLayer, Kleros, UMA; 11 off-chain agents by majority); resolves when X children agree, per an aggregation rule referenced in the adjudication definition. Design direction: a child unresolved after a window is excluded, shrinking the pool but never the threshold, and if the threshold becomes unreachable the panel resolves with a no-quorum outcome defined in the definition's outcome space, so panel failure is a legitimate resolution the consumer can act on. How non-conforming child resolutions are handled, and how operator independence across the Y children is assessed, are implementation concerns. Detailed semantics come with the implementation.
4. **No-adjudication passthrough**: the uncontested path. If nobody contests within a window, the passthrough itself activates and resolves the case without invoking any child, performing both transitions in one transaction (the core lifecycle permits no skipped states, so `Active` is passed through and each event is emitted). If contested, it registers and activates a child on the configured Adjudicator. Lets integrators wire the standard in everywhere while paying for adjudication only on actual contests.

## Part 3: Leaf adjudicators (v0.1 reference implementations)

Composites need leaves. v0.1 implements at least the following leaf Adjudicators, factory-instantiated like everything else:

1. **GenLayer adjudicator**: adapter fronting GenLayer. An intelligent contract decides the case; the adapter surfaces its resolution.
2. **Kleros adjudicator**: adapter fronting Kleros. Activation starts the underlying Kleros case, forwarding the arbitration cost in the same transaction; the adapter stays `Active` through any internal appeal rounds and surfaces the final outcome as the resolution.
3. **Wallet adjudicator**: a single wallet address is authorized to resolve. The adjudicator itself is fully on-chain; what operates the wallet (an off-chain agent, a human, a multisig) is out of scope, which is why it is named for the wallet, not the operator.
4. **Echo adjudicator (testing)**: resolves on activation with a canned resolution taken from its adjudication definition, whose content includes or references the resolution attestation UID to echo. The resolution never travels through `data` (the core bars semantic content there); the echo reads it from the on-chain definition at registration, which also exercises the definition-reading path end-to-end. Honors the full `Registered → Active → Resolved` lifecycle, with no decision logic at all: a test double for exercising Adjudicables, composites, and integrations without any real system behind it.

## Part 4: What the factory standard itself covers

- **Instantiation**: one factory call per catalog entry, parameterized by child Adjudicator addresses and an EAS configuration attestation; returns the deployed composite's address.
- **Discovery**: a standard `CompositeDeployed(factory, composite, configUID)` event and enumeration of deployed instances, so indexers and UIs find all standard-conformant composites. ("Deployed" deliberately: contract instantiation vocabulary stays distinct from the case lifecycle's register/activate.)
- **Configuration via EAS**: composite parameters (children, windows, thresholds) referenced as an attestation, consistent with the core ERC's semantic layer.

## Future work (not in v0.1)

- **Escalation funding patterns**: pay-as-you-go (the escalating party pays the next round's cost at escalation time, through the router's `payable` entry point) vs. pre-funded budget (worst-case cost escrowed at registration, refunded if unused, with a top-up path for dynamic downstream pricing). Deferred; v0.1 leaves escalation funding implementation-defined.
- **The last-mover problem in panels** (known, unsolved for now): child resolutions are public the moment they land, so in any split the not-yet-resolved child knows it holds the casting vote, making it the natural target for bribery or extortion. Commit-reveal across children is impossible while resolutions are on-chain verified attestations. A future adjudicator variant with committed or encrypted resolutions and a reveal phase could address it; until then, child-selection diligence (children whose internal processes already resist targeted corruption) is the mitigation.

## Relationship to the core ERC

Everything here is additive. A composite is a full Adjudicator: consumers use `registerAdjudication` / `activate` / `status` / `resolution` and never need to know they are talking to a tree. The core ERC ships without any of this; this ERC can iterate at its own pace (new composites, new funding patterns) without touching the core interface.
