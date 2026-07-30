# Knowledge Base: Composable Adjudication ERC

Public background information needed to write the standard and its reference implementations. **Nothing here is normative.** These are implementation details of specific systems and platform facts that inform design decisions but must never appear in the ERC text itself.

## GenLayer

- GenLayer runs on its own EVM-compatible L2 built on the ZKSync stack. Intelligent contracts (Python, executed in GenVM by validators with LLM and web access, under Optimistic Democracy consensus) are callable natively from EVM contracts on that chain; no bridging is needed when the Adjudicator is deployed there.
- Intelligent contracts can perform consensus-verified reads of external chains (each validator does an `eth_call` via RPC; agreement via `strict_eq`), so a GenLayer contract can independently verify state on another chain (e.g., that an adjudication was activated and paid) rather than trusting a relayer's claim.
- An intelligent contract cannot be invoked synchronously from another chain. Cross-chain flows are asynchronous: a relayer/agent watches events and submits GenLayer transactions, or the intelligent contract pulls external state; results return via an authorized wallet or bridge message.
- GenLayer-side costs (GEN gas, LLM inference) are paid by whoever sends GenLayer transactions, typically the adapter operator, who recoups from fees collected on the EVM side. This conversion is operational, not protocol.
- Intelligent contracts support on-chain factories (`gl.deploy_contract`), so per-case worker contracts are possible GenLayer-side.

## ZKSync-stack cross-chain mechanics

- **L1 → L2 priority transactions:** a contract on L1 can programmatically request a transaction on the L2 carrying both calldata and value (via the bridgehub). Asynchronous (typically minutes); the caller pre-pays L2 gas as part of the L1 value.
- **L2 → L1 messaging:** proof-based, trust-minimized, but waits on proof finalization (hours). Fast paths use an authorized relayer with the native message as fallback.

## Kleros

- `createDispute` requires the arbitration cost (ETH) in the same transaction. This is why the standard's activation channel must be `payable`.
- Arbitration cost is dynamic: it depends on court, juror count, and conditions at dispute time. Quotes go stale; pay-at-escalation beats pre-funding.
- Internal appeal rounds exist; from the standard's perspective a Kleros adapter simply stays `Active` until the outcome is final.
- Kleros operates cross-chain today via home/foreign proxy contract pairs; Kleros v2 lives on Arbitrum. An adapter is essentially a standardized wrapper over existing proxy practice.
- Prior art interfaces: ERC-792 (`extraData` pattern for court/juror parameters) and ERC-1497 (evidence; implementations deviated significantly, the cautionary tale behind "standardize state and pointers, not behavior").

## UMA

- Optimistic oracle model: an assertion is posted with a bond and stands unless challenged within a window. This maps naturally to a leaf whose internal flow is assert, challenge-window, final, surfacing as `Active` until final.
- Bonds are ERC-20-denominated: adapters handle them via approve/transferFrom choreography inside the standard's `payable` calls; no interface change needed.

## EAS (Ethereum Attestation Service)

- Attestations can be **on-chain** (readable from Solidity via `getAttestation`) or **off-chain** (signed, zero gas, verifiable, but unreadable from Solidity). This asymmetry is why any parameter a contract must enforce needs an on-chain source, and why the standard now requires definition attestations to be on-chain on the Adjudicator's chain.
- Attestations reference other attestations via `refUID` and schema fields; this is how a definition carries or references the baseline agreement.
- Attestations carry `expirationTime` and revocability flags; the standard constrains both (zero expiration, irrevocable schemas) for definitions and resolutions.
- One deployed schema registry per chain; schemas are referenced by UID.
- EAS itself is a singleton holding all attestations: precedent for the registry-style (ID-based) design over contract-per-case.

## EVM platform facts used in design decisions

- **Zero-slot sentinel:** an unwritten storage slot reads as zero, so a status enum's zero value must mean "does not exist" (`None`).
- **Storage scales flat:** reading or writing a mapping slot costs the same regardless of how many entries the mapping has; there is no "contract getting full" (the ~24KB cap is on code, not storage). Many small contracts consume *more* total state than one mapping (account entries plus code per contract).
- **Deployment vs. storage costs:** registering an ID costs a few storage writes (tens of thousands of gas); deploying even a minimal EIP-1167 proxy adds ~45k gas plus per-call delegatecall overhead; full bytecode deployment costs hundreds of thousands. Relevant to registry-vs-per-case and to composites registering several children per case.
- **Constructors are outside the standardizable surface:** not part of the runtime ABI, not advertisable via ERC-165, not introspectable after deployment. This is why configuration binding is a factory/implementation concern, not core-ERC surface.
- **Singleton precedent:** EAS, ERC-4337 EntryPoint, Uniswap v4, Kleros v1. The ecosystem trend is toward ID-based singletons for gas efficiency; per-entity contracts persist where entities are long-lived and asset-holding (Safe, Uniswap v2 pairs), not ephemeral cases.
