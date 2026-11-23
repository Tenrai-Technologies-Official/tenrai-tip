# TIP draft: Private transfers (TSP-2)

| Field | Value |
| --- | --- |
| TIP | Unassigned |
| Title | Private transfers (TSP-2) |
| Author | Tenrai Technologies |
| Status | Draft |
| Type | Consensus |

## Abstract

TSP-2 adds private notes beside transparent UTXOs. Value enters the private notes with an ENTER, moves between them with a TRANSFER and leaves them with an EXIT. Every statement carries a zero-knowledge proof (Q1, a Circle-STARK over M31 with SHAKE256-384) that the node verifies under the pinned key of its circuit. A cut executes private statements after its transparent work, writes their notes, nullifiers and outputs in the same install as its state, and keeps the issued-value equation exact. The authenticated network profile determines activation. The current mainnet configuration includes the private profile from genesis; subsequent changes use an explicitly scheduled rule-set upgrade.

## Motivation

Every transparent output publishes its owner and amount. Private notes hide who pays whom and how much, while the chain still proves that no value is created and no note is spent twice. The transparent side stays as it is: ENTER and EXIT are the only bridges, and both are proven.

## Specification

### Classes and circuits

| Class | Spends | Creates | Circuit |
| --- | --- | --- | --- |
| ENTER | Transparent outputs, up to the transaction input limit, signed as a `PrivateEnter` transaction | 1 or 2 notes | `enter-1`, `enter-2` |
| TRANSFER | 2 notes | 2 notes | `transfer` |
| EXIT | 1 or 2 notes | 1 transparent output | `exit-1`, `exit-2` |

Each circuit has a verifier key pinned in the node. Its 48-byte digest is SHAKE256 over the ASCII string `TSP2/NATIVE-GATE/VERIFIER-KEY` followed by the canonical key bytes. The full digests, in the order below, form the `TRAI_TSP2_VERIFIER_KEYS_V1` domain hash. The rule set binds that commitment through its private profile. Changing a key requires a new rule set.

| Circuit | Verifier key digest (first 8 bytes) |
| --- | --- |
| `transfer` | `6f5671c87149a0c9` |
| `enter-1` | `9e14d74ce8df52ae` |
| `enter-2` | `dcedff47dc277fda` |
| `exit-1` | `44dc7a7aa08a630d` |
| `exit-2` | `a2b0d9aed41f3b12` |

### Statements

Every statement starts with the same 412 bytes: statement version, protocol version, chain ID, rules hash, suite binding digest, resource schedule digest, validity window, fee, intent nonce and the anchor (height, cut ID, state root, private-state root). An ENTER continues with the entered amount, the digest of its input outpoints and per output its note commitment and capsule digest; an EXIT with the amount, the destination lock and its nullifiers in ascending order. The statement digest is the proof's public input. The byte layouts are in `protocol/tsp2-enter-exit.md` of the node repository; the transfer statement is in `protocol/tsp2-zk-v1.md`.

The anchor is a finalized cut of the same chain. A statement is valid from `valid_from` to `valid_until` inclusive.

### Frames

A TRANSFER frame contains its statement, two capsules and a proof, up to 1,103,546 bytes under the release profile. An ENTER frame contains its statement, the canonical signed `PrivateEnter` transaction, one 1,420-byte capsule per output and a proof. An EXIT frame contains its statement and proof. Each class uses its own wire limit and canonical encoding. The proof limit is 1,100,000 bytes. Authenticated private status reports the active limits.

### Admission

A node checks the cheap parts before any proof work: a known statement, a pooled or final nullifier, a pooled ENTER input, the pool's budget. Then:

- ENTER: the `PrivateEnter` transaction is authorized like any transaction, spends unspent outputs whose values sum to the entered amount, and names the statement digest; its inputs digest and the statement match; the fee meets the schedule; the proof verifies under the ENTER key of its output count.
- TRANSFER: its nullifiers are new; the fee meets the schedule; the proof verifies under the transfer key.
- EXIT: its nullifiers are new; the destination is a canonical lock; the proof verifies under the EXIT key of its source count.

A recipient registration combines the owner's ML-DSA-65 authorization with a signed `PrivateRegister` funding transaction bound to that authorization. A cut applies the funding and publishes the recipient record atomically. The registry accepts one record per owner.

### Execution

One cut executes at most four private statements of all classes together and at most 16 registrations. It runs them after its transparent transactions, in this order: registrations by registration ID, then transfers, ENTERs and EXITs, each class by statement digest. The order follows from the items alone, so the order in which a node received them never reaches the state; which items a proposer takes is its own choice. No two statements of a cut share a nullifier, a note commitment or a capsule digest, and an ENTER's inputs must still be unspent after the cut's transparent work; otherwise the cut is invalid.

- ENTER removes its inputs from the UTXO set and appends its notes.
- TRANSFER inserts its nullifiers and appends its notes.
- EXIT inserts its nullifiers and creates one UTXO at the outpoint (statement digest, 0) with the source `private_exit`, or burns the amount if the destination is the burn lock.

The private value total grows by the notes of an ENTER and shrinks by amount plus fee of an EXIT and by the fee of a transfer. Every fee is split as a transfer fee: half is burned, half goes to the stake pool. The cut's supply check includes the private total: transparent outputs, private notes, bonded stake, reserves and burns add up to the minted supply.

A cut with private statements commits to their digests in execution order, bound to the execution ID of its work. The private state (note log, recipient registry, nullifier set and the private value total) enters the cut's state commitment as its compute root.

### Propagation

A proposal names the items of its cut in a section of its envelope that only cuts with items have: registration IDs and statement IDs in execution order. Voters and followers run exactly those items, each verified by themselves, and fetch what they lack from their peers; nodes announce what their pool holds and serve what they ran (P2P wire version 10). A node that lacks an item waits for it instead of voting. Only a malformed item, one that is not what it was sent as, or a failed proof or signature counts against a peer.

Items bind the rules they were made under. A cut whose rules differ from its predecessor's runs no private item; a node refuses items made under other rules than its next cut runs and drops them from its pool when the rules change.

### Storage and sync

A cut stores, in the same write as its state, the rows wallets read (notes with their capsules, recipient records, nullifiers by height) and the canonical bytes of every item it ran. Nodes keep the private values of the last 1,800 cuts, the longest a statement may wait after its anchor; archive nodes keep every state; pruned nodes drop the items with the bodies below their history floor and keep the rows. A snapshot carries the private state and the cuts of that window, so a node started from it verifies statements anchored below it. A node that lacks rows a wallet asks for answers 404 instead of an empty page.

### Activation

A rule set runs private transfers only if its private profile is set. The current mainnet configuration sets that profile from genesis. Testnet activation remains governed by its own compiled profile; a development chain can explicitly schedule `development-private`. An inactive profile keeps the disabled compute root and rejects private items. A later activation or profile change requires the rule set and height named by a release, with node operators following that schedule.

Wallets enable private operations only when authenticated status reports `active: true`, `protocol.fits_consensus_limits: true` and `protocol.security_qualification` of `internally_qualified` or `complete`. The `complete` state includes independent review.

## Rationale

- **Proofs on both bridges.** A public deposit would publish the split of an ENTER and tie it to a recipient record; a proven ENTER hides both, and a proven EXIT hides which notes paid.
- **One circuit per shape.** Fixed circuits without switches keep every constraint active; five keys cost five pinned files and nothing at run time.
- **Four statements per cut.** The cap bounds private proof verification work per cut.
- **Fees at the schedule's limits.** Wallets pay the fee of the largest allowed statement, so a fee never depends on the size a proof turns out to have.
- **Order from the items alone.** Registrations by ID and statements by class and digest give every node the same order without trusting anyone's arrival order. Position in a cut buys nothing, two conflicting statements make a cut invalid as a whole, and replay protection rests on nullifiers, spent inputs and the registry, not on order.
- **Nothing crosses a rule change.** An item proven or signed for one rule set never runs under another, so an upgrade has no queue to drain.

## Backward compatibility

A chain whose genesis profile is inactive retains the disabled compute root until an explicit rule-set activation. The current mainnet genesis includes the private profile, so every participating mainnet node must support it from height zero. Nodes that do not know that profile refuse the chain. A later profile change is a coordinated upgrade. Wallets and explorers see two new values in the API: the UTXO source `private_exit` and the transaction kind `private_enter`.

## Security considerations

- **Denial of service.** Proof verification is the expensive step; it runs last, under pool budgets of 64 statements, 80 MiB and 256 registrations, and a statement refused at admission never enters the pool. Frames above the wire limit are refused before parsing.
- **Double spends.** Nullifiers are checked at admission, again at selection and once more in execution against the cut's own statements; ENTER inputs are checked against the cut's transparent work.
- **Linkability.** An EXIT's amount and time are public; wallets exit from notes a transfer produced and avoid the exact amount of a recent ENTER.
- **Resource accounting.** Registrations and statements are subject to compiled pool, per-cut and storage budgets.

## Implementation and testing

The node implementation uses `tenrai-traidag`, `tenrai-tsp2-verifier`,
`tenrai-tsp2-prove`, `tenrai-traidag-storage` and `tenraid`. The
`tsp2-*.json` files in `tenrai-test-vectors` specify reference inputs.
Integration checks cover all three statement classes, issued value across cuts,
and state recovery after a restart or from a snapshot.
Wallet conventions are described in the [private accounts draft](https://github.com/Tenrai-Technologies-Official/tenrai-tcc/blob/main/tcc-draft-private-accounts.md).
