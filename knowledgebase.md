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

Verified 2026-07-30 against `eas-contracts` master (Semver 1.4.0) and the Ethereum mainnet deployment `0xA1207F3BBa224E2c9c3c6D5aF63D0eb1582Ce587`; deployed v0.26 and current master agree on every mechanism below.

**On-chain vs off-chain**

- Attestations can be **on-chain** (stored in EAS's `_db` mapping, readable from Solidity via `getAttestation`) or **off-chain** (EIP-712-signed payloads distributed off-chain, zero gas, but *unreadable from Solidity*: `getAttestation` on an off-chain UID returns a zeroed struct, indistinguishable from "never existed"). The only on-chain hooks for off-chain attestations are `timestamp(bytes32)` and `revokeOffchain(bytes32)`, which record a Unix timestamp against an arbitrary hash and nothing else. This asymmetry is why any parameter a contract must enforce (windows, thresholds, routing) needs an on-chain source (instantiation config, constructor, or call `data`), and why this standard requires definition attestations to be on-chain (decision 16), giving up off-chain definitions' zero gas cost for contract-readable definitions.
- `getAttestation(uid)` never reverts: an unknown UID yields a zeroed struct. `isAttestationValid(uid)` exists precisely because of that and checks only `uid != 0`. Consumers must test existence explicitly.

**Attestation record**

- Struct: `uid, schema, time, expirationTime, revocationTime, refUID, recipient, attester, revocable, data`. `schema` is the schema UID, so a contract can gate on `attestation.schema == EXPECTED_UID`, a plain `bytes32` comparison with no registry lookup.
- **Irrevocability is protocol-enforced, not advisory.** `_attest` reverts `Irrevocable()` when the schema's `revocable` flag is false and the request asks for a revocable attestation; `_revoke` re-checks. A schema's `revocable` flag is immutable once registered. Registering a schema irrevocable is therefore a hard guarantee for every attestation ever made under it.
- **`expirationTime` is validated only at attest time** (must be zero or strictly in the future) and is **not** enforced afterwards: `isAttestationValid` checks existence only, and `getAttestation` returns an expired attestation unchanged. Expiry is purely informational: each consumer must compare it to `block.timestamp` itself. EAS has no "validity" notion that expiry feeds into.
- `refUID` is validated to point at an existing attestation on the same EAS instance (`NotFound()` otherwise). That is its *only* protocol semantic: no schema-compatibility check, no revocation or expiry check on the referent, no limit on how many attestations share a `refUID`. It is nonetheless the mechanism by which a definition references the baseline agreement, so that link is conventional rather than protocol-enforced.
- **Attestation UIDs are not globally unique across chains.** The preimage is `schema, recipient, attester, time, expirationTime, revocable, refUID, data, bump`, with no chain id and no EAS contract address. Two chains can hold the same UID for unrelated attestations; a bare 32-byte UID is unique only within one EAS deployment's storage.
- EAS stores `data` as opaque bytes and does not validate it against the schema string. (Inferred from the absence of any such check plus the existence of resolvers for exactly this purpose, not separately confirmed.)

**Schema registry**

- **Schema UID = `keccak256(abi.encodePacked(schemaString, resolver, revocable))`**, with no chain id, no registry address, no sender, no nonce. The identical triple yields the identical UID on every chain, so a canonical schema UID *can* be named as a chain-independent constant, but only with `resolver = address(0)`, since a resolver deployed at different addresses per chain diverges the UID.
- Schema strings are stored **verbatim**; the registry parses and validates nothing. Whitespace is significant to the UID (`"uint256 a,bool b"` is not `"uint256 a, bool b"`), field names are optional, and neither the SDK nor `easctl` normalizes before registering. Anything publishing a canonical schema must publish byte-exact strings.
- `SchemaRegistry.getSchema(uid)` is `external view` returning `{uid, resolver, revocable, schema}`, so the schema string is genuinely readable on-chain, roughly 10-20k gas cold for ~100 bytes.
- Registration is not idempotent: re-registering an identical triple reverts `AlreadyExists`.
- **No versioning primitive.** Any change to the schema string is a new, unlinked UID; evolution is a social/tooling convention (typically a referencing attestation), never a protocol feature.
- Resolvers receive the full `Attestation` (including `data`) at attest and revoke time, can veto by returning false, and may be payable, so a resolver *can* enforce field-level validity, at the cost of the chain-independent UID above.

**Deployment**

- EAS and SchemaRegistry are **not** at one address across chains: mainnet `0xA1207F3B…` / `0xA7b39296…`; OP-stack chains (Optimism, Base, Blast) share the predeploys `0x42…0021` / `0x42…0020`; Arbitrum, Polygon, Scroll, zkSync, Celo, Linea and others each differ. No universal constant exists: an Adjudicator must take its EAS address as deployment configuration. Official list: `docs/quick--start/contracts.md` in `eas-docs-site`.
- One deployed schema registry per chain; schemas are referenced by UID.
- EAS itself is a singleton holding all attestations: precedent for the registry-style (ID-based) design over contract-per-case.

## EVM platform facts used in design decisions

- **Zero-slot sentinel:** an unwritten storage slot reads as zero, so a status enum's zero value must mean "does not exist" (`None`).
- **Storage scales flat:** reading or writing a mapping slot costs the same regardless of how many entries the mapping has; there is no "contract getting full" (the ~24KB cap is on code, not storage). Many small contracts consume *more* total state than one mapping (account entries plus code per contract).
- **Deployment vs. storage costs:** registering an ID costs a few storage writes (tens of thousands of gas); deploying even a minimal EIP-1167 proxy adds ~45k gas plus per-call delegatecall overhead; full bytecode deployment costs hundreds of thousands. Relevant to registry-vs-per-case and to composites registering several children per case.
- **Partial ABI decoding is unsafe over untrusted bytes when a dynamic field leads.** Verified empirically (Foundry, solc 0.8.30) plus the ABI spec. Decoding a 2-field prefix of a 7-field encoding *does* work, and Solidity's decoder does not reject trailing data, but the spec calls that current behavior, not a guarantee ("The Solidity ABI decoder currently does not enforce strict mode"). Two sharper problems: (1) a too-short payload reverts reliably **only** for all-static tuples, since with a leading `address[]` a missing trailing static field silently aliases the array's length word instead of reverting; (2) head slots hold attacker-controlled offsets, so a crafted payload can make a 2-field prefix decode return *different* values than a full decode of the same bytes. An all-static leading prefix has neither problem: static fields are read from fixed head slots with no offset-chasing, and a short payload always reverts.
- **Constructors are outside the standardizable surface:** not part of the runtime ABI, not advertisable via ERC-165, not introspectable after deployment. This is why configuration binding is a factory/implementation concern, not core-ERC surface.
- **Singleton precedent:** EAS, ERC-4337 EntryPoint, Uniswap v4, Kleros v1. The ecosystem trend is toward ID-based singletons for gas efficiency; per-entity contracts persist where entities are long-lived and asset-holding (Safe, Uniswap v2 pairs), not ephemeral cases.
