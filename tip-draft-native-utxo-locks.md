# TIP draft: Native UTXO conditions V1

| Field | Value |
| --- | --- |
| TIP | Unassigned |
| Title | Native UTXO conditions V1 |
| Author | Tenrai Technologies |
| Status | Draft |
| Type | Consensus |

This proposal specifies timelocks, hashlocks and HTLCs under protocol version 2.
Activation follows the network's rule schedule.

## Lock and descriptor

Lock41 authentication profile `3` commits to a bounded native condition. Direct
and threshold profiles `1` and `2` retain their exact encodings. The 40-byte commitment is
`SHAKE256("TRAI_NATIVE_LOCK_V1" || descriptor)[0..40]`. A descriptor starts with
version `0x01`, followed by one opcode and its operands:

| Opcode | Operands in canonical order | Length including header |
| --- | --- | ---: |
| `0x01` timelock | owner Lock41, unlock height u64 | 51 bytes |
| `0x02` hashlock | owner Lock41, SHA-256 digest (32 bytes) | 75 bytes |
| `0x03` HTLC | claim Lock41, refund Lock41, SHA-256 digest, refund height u64 | 124 bytes |

Every integer uses unsigned big endian encoding. Heights are nonzero finalized
cut heights. Each owner must use direct ML-DSA-65 profile `1`; nested conditions,
threshold owners, unknown versions/opcodes, trailing bytes and variable-width
integer encodings are rejected. Future contract versions require explicitly
specified consensus rules; unknown programs never become anyone-can-spend.

The descriptor is revealed when spending, so its creator must preserve and
share it with the intended recipient. An address commits to a descriptor but
does not contain enough data to reconstruct it. Wallets must export/import
descriptors and watch their separate native addresses. Ordinary key-address
scanning cannot discover all such deposits.

## Witness and authorization

Witness profile `5` is encoded as:

`0x05 || descriptor_length:u16 || descriptor || branch:u8 ||
preimage_length:u8 || preimage || public_key:1952 || signature:3309`

Descriptor length is at most 124 bytes; preimage length is at most 64 bytes.
Branch `0` means claim; branch `1` means refund. Timelocks and refund witnesses
require an empty preimage; standalone hashlock claims require 1 through 64 bytes,
and HTLC claims require **exactly 32 bytes**. Exactly one
ML-DSA-65 verification is charged in addition to all serialized-byte costs.
Native outputs and native witnesses require transaction protocol version >=2.

For protocol versions >=2 the consensus parameter hash also commits to
`SHA-384` with domain `TRAI_NATIVE_LOCK_RULES_V1` and the standard length framing,
over these ten unsigned u16 big endian fields:
`[1, 1, 1, 2, 3, 3, 5, 124, 64, 32]` (rule version, descriptor version,
three opcodes, lock profile, witness profile, descriptor bound, general preimage
bound, HTLC preimage size). Protocol-1 parameter hashing excludes this commitment.

The descriptor must reproduce the **spent UTXO's outer native lock**, and its
chosen owner must reproduce the witness's public-key commitment. The signature
uses the existing `TRAI/UTXO/SPEND/1` context over the standard spend message,
binding chain, protocol, entire body, input index, anchored amount and outer
lock. A revealed secret alone cannot authorize a transfer or redirect funds.
The transaction identity remains witness-free; witness identity and work-body
commitments bind the full witness, descriptor and preimage.

## Height rules

Let `[from, until]` be the signed transaction's inclusive validity interval.

| Condition and branch | Required interval and secret |
| --- | --- |
| Timelock claim | `from >= unlock_height`, empty preimage |
| Hashlock claim | `SHA256(preimage) == digest` |
| HTLC claim | `until < refund_height`, matching 32-byte preimage |
| HTLC refund | `from >= refund_height`, empty preimage |

Empty intervals are invalid. Existing transaction validity-window limits still
apply. The entire signed interval must satisfy the condition: historical work
admission does not authorize executing a claim after its timeout. Actual cut
execution continues to enforce the interval at its finalized execution height.
Claim and refund intervals are disjoint, including exactly at the refund height.

No wall clock, floating point value, local arrival order, platform-sized wire
integer, timezone or network-latency estimate enters these decisions. A height
is not a promise of a particular elapsed duration: chain progress can slow.

## Value conservation and supported roles

Native conditions use the existing UTXO membership, ordering, uniqueness,
snapshot anchoring, checked amount accounting and cut execution mechanisms.
They do not introduce another minting or balance path. Successful spends consume
the same outpoints as ordinary payments; competing claim/refund transactions
cannot create independent copies of the coin. Failed, premature or expired
transactions must leave both UTXO and key-registry state unchanged.

Native locks are transparent payment-output conditions. Native locks in genesis
allocations, mining rewards, validator/stake owner/reward roles and generic
message authorization are rejected in this revision. TSP-2 authorization must
reject native locks under these rules.

## Atomic-swap interoperability

The digest is ordinary SHA-256 of the raw preimage, without a Tenrai prefix,
so the same secret can satisfy a counterparty chain's SHA-256 HTLC. The
counterparty transaction format, signature scheme and clock are chain-specific.
An atomic swap still needs independently verified funding/finality, sufficient
timeout separation, and monitoring for the public preimage. Generate a random
32-byte secret; broadcasting a claim publicly reveals it. Never save a secret
in a descriptor, ordinary funding payload, application logs or watch storage.

References: [BIP 65](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki),
[BIP 112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki),
and the HTLC discussion in [BIP 199](https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki).
Tenrai specifies its own height and interval rules above; it does not execute
Bitcoin Script or claim byte-for-byte Script compatibility.

## Implementation and testing

The node's native lock verifier follows `protocol/native-utxo-locks-v1.md`.
The `native-locks-v1.json` vectors in `tenrai-test-vectors`
cover descriptor encoding, commitments and witness authorization. Interoperability
checks cover activation and refund boundaries, value conservation, fees and
consistent rejection of malformed descriptors and witnesses.
