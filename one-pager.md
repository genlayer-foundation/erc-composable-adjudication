# Composable Adjudication — One Pager

## The problem

Every protocol that needs decisions resolved — escrows, marketplaces, prediction markets, insurance, agent-to-agent commerce — integrates each adjudication system bespoke. GenLayer, Kleros, and UMA each expose a different interface, so a project needing both deterministic resolutions (a price feed) and subjective ones (was the work delivered?) must implement and maintain several integrations. There is no way to combine systems, and no way to start cheap and escalate only when contested.

## The idea: one interface for all adjudication

A minimal standard interface — the **Adjudicator** — behind which any resolution system can sit: GenLayer validators, Kleros jurors, UMA's optimistic oracle, or a single off-chain agent with one wallet. Consumers (**Adjudicables**) integrate once. An adjudication is **registered** with a pointer to its definition — typically at agreement time, before any contest, like an arbitration clause — and **activated**, with the contest evidence attached, only if a decision is actually needed; it then runs to completion and exposes a resolution. Existing systems join via thin adapter contracts; no changes required on their side.

## Escalation: mix and match

Adjudication systems differ in cost and in trust assumptions: an AI-validator resolution (GenLayer) is cheap — a single off-chain agent cheaper still; crypto-economic juries and optimistic oracles (Kleros, UMA) cost more, and their jurors can in principle be bribed; real-world alternative dispute resolution (ADR) is the costliest of all. Which is "more secure" is case-dependent and contestable — so the standard doesn't rank systems or fix an order. It provides the building blocks; whoever composes an escalation chain chooses which systems sit on it and in what sequence, escalating on contest. Later rungs discipline earlier ones by their mere availability. Each stage is funded only when it is used — activation pays for the first round; how later rounds are funded is each composite's design.

## Composability: adjudicators all the way down

An aggregate of adjudicators **is** an adjudicator. An escalation router (GenLayer → Kleros) implements the same interface as its children. So does an X-of-Y panel — 2-of-3 across GenLayer, Kleros, and UMA, or a majority of 11 off-chain agents. Because compositions are themselves adjudicators, they nest arbitrarily, and consumers never need to know whether they are talking to a single court or a whole tree of them.

## What the standard covers — and what it doesn't

**Standardized:** a two-phase lifecycle — registration at agreement time (cheap, dormant, one canonical case ID from day one) and `payable` activation at contest time (so fee-charging systems work) — four states (`None / Registered / Active / Resolved`), pull-based resolution reading, interface discovery via ERC-165, and events. **Delegated to EAS attestations:** adjudication definitions, evidence, and resolutions — the standard carries pointers, schemas live in an EAS registry, keeping the on-chain surface minimal and the semantics evolvable. **Out of scope:** fee amounts and semantics, evidence timing windows, and escalation entry points — all implementation-defined.

## Two ERCs

1. **Core** — the Adjudicator interface. Tiny, generic, designed from zero; existing systems join via adapters.
2. **Composition & factories** — composite introspection (walking the tree of child adjudications) and pre-built, instantiate-from-factory glue for common setups: single adjudicator, GenLayer → Kleros escalation, X-of-Y panels, no-adjudication passthrough.

## v0.1 sample use cases

- Binary resolution on a single system (GenLayer only; Kleros only)
- GenLayer → Kleros escalation
- X-of-Y round of off-chain agents
- No contest → no adjudication executed; contest → the standard kicks in

## Next steps (technical)

Circulate this draft with co-authors; converge on the EAS schemas and `registerAdjudication`/`activate` parameters; build the v0.1 reference implementation and adapters.
