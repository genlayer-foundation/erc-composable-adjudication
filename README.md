# Composable Adjudication ERC

A draft Ethereum standard for **composable adjudication**: one minimal interface through which any on-chain contract can request and consume decisions from any adjudication system — with escalation on contest and arbitrary composition of systems.

## Why

Every protocol that needs decisions resolved — escrows, marketplaces, prediction markets, insurance, agent-to-agent commerce — integrates each adjudication system bespoke. GenLayer, Kleros, and UMA each expose a different interface, so a project needing both deterministic resolutions (a price feed) and subjective ones (was the work delivered?) must implement and maintain several integrations. There is no way to combine systems, and no way to start cheap and escalate only when contested.

## The idea

A minimal standard interface — the **Adjudicator** — behind which any resolution system can sit: GenLayer validators, Kleros jurors, UMA's optimistic oracle, or a single agent operating a wallet. Consumers (**Adjudicables**) integrate once. An adjudication is **registered** with a pointer to its definition — typically at agreement time, before any contest, like an arbitration clause — and **activated**, with the contest evidence attached, only if a decision is actually needed. Existing systems join via thin adapter contracts; no changes required on their side.

Two properties carry the design:

- **Escalation, mix and match.** Adjudication systems differ in cost and in trust assumptions; which is "more secure" is case-dependent and contestable, so the standard doesn't rank systems or fix an order. It provides the building blocks — whoever composes an escalation chain chooses which systems sit on it and in what sequence, escalating on contest. Later rungs discipline earlier ones by their mere availability, and each stage is funded only when it is used.
- **Composability, adjudicators all the way down.** An aggregate of adjudicators *is* an adjudicator: an escalation router implements the same interface as its children, and so does an X-of-Y panel. Compositions nest arbitrarily, and consumers never need to know whether they are talking to a single court or a whole tree of them.

The standard keeps the on-chain surface tiny — a two-phase lifecycle (`None / Registered / Active / Resolved`), payable entry points, pull-based resolution reading, ERC-165 discovery — and delegates all semantics (definitions, evidence, resolutions) to EAS attestations, so meaning can evolve without touching deployed contracts. Fee semantics, evidence windows, and escalation entry points are deliberately implementation-defined.

## Repository contents

| Document | What it is |
|---|---|
| [`one-pager.md`](one-pager.md) | The pitch: problem, idea, escalation, composability |
| [`erc-composable-adjudication-draft.md`](erc-composable-adjudication-draft.md) | Natural-language draft of ERC #1 — the core Adjudicator interface |
| [`erc-adjudication-factories-outline.md`](erc-adjudication-factories-outline.md) | Idea outline for ERC #2 — composition, factories, and reference leaf adjudicators |
| [`design-decisions-needed.md`](design-decisions-needed.md) | Open design decisions still to be made |
| [`knowledgebase.md`](knowledgebase.md) | Non-normative background on the systems and platform facts informing the design |
| [`AGENTS.md`](AGENTS.md) | Project context: settled design decisions and working conventions |

## Status

Pre-draft, natural-language specification — no reference implementation yet. See [`design-decisions-needed.md`](design-decisions-needed.md) for what remains open.
