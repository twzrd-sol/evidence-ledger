# evidence-ledger (Pinocchio)

Append-only on-chain registry of merkle roots over TWZRD's signed evidence
leaves. A published entry means one thing: *TWZRD's log key signed this head,
and it was anchored before slot S.* It is not a vouch, an eligibility flag, or
an unlock. The program has no claim, mint, transfer, or scoring instruction and
no per-seller account, by construction.

Program id: `BzBAYJxUtJp6mUkJPjEYjd8vdb2FUGnAfB5X9LqrQ72W` (the program keypair is not in this repository).

This repository is the public copy of the program's source, published 2026-10-13. The source here is
byte-identical to the revision deployed on Solana mainnet (last changed 2026-09-04 in TWZRD's private
development monorepo, deployed 2026-09-07). The `security.txt` embedded in the deployed program still
points `policy` and `source_code` at that private repository; this repository is the readable copy.
Security contact: `security@twzrd.xyz`.

## Accounts

| Account | PDA | Size | Notes |
|---|---|---|---|
| `Ledger` | `["ledger", keccak256(log_id)]` | 128 | one per log / leaf domain; `scheme`, `authority`, immutable `trusted_signer`, strictly-increasing cursor |
| `RootEntry` | `["root", ledger, seq_le]` | 152 | one per anchored head; never rewritten (a correction is a later seq) |

Schemes: `1` = sorted-pair keccak batch root (odd trailing node paired with itself), `2` = RFC 9162 keccak log
(the `twzrd_agent_intel.log_rfc6962` module in the twzrd-agent-intel package). Layouts and discriminators (`sha256("account:<Name>")[..8]`)
are in `src/state.rs`.

## Instructions (`sha256("global:<name>")[..8]` discriminators)

- `init_ledger(authority, trusted_signer, scheme, log_id)` — the chain hashes `log_id` into the seed.
- `set_authority(new)` — who may spend to publish. The trusted signer has no rotation path on
  purpose: the log spec says a key change means a new `log_id`, never a re-signed head.
- `publish_root(seq, root, manifest_hash, leaf_count)` — scheme 1 only.
- `anchor_head(tree_size, timestamp, root, log_id)` — scheme 2. Requires the immediately
  preceding instruction to be one ed25519 precompile call whose (pubkey, message) is exactly
  (`trusted_signer`, the STH preimage rebuilt on-chain). The program never signs, so the
  anchoring key cannot manufacture a head the log key never issued.
- `verify_inclusion(seq, leaf, proof)` — no signer, no writes; CPI-callable. Scheme 2 takes
  `(index, path)` per RFC 9162.

Errors: `6000` Unauthorized, `6001` AlreadyInitialized, `6002` SequenceMismatch, `6003` InvalidProof,
`6004` InvalidPubkey, `6005` WrongScheme, `6006` MissingSignatureCheck, `6007` SignatureBindingMismatch.

## Build and test

```bash
cargo test --release                                   # unit: keccak, layouts, both merkle schemes (Python reference vectors)
cargo build-sbf                                         # target/deploy/evidence_ledger.so
cargo test --release --features localtest --test litesvm_ledger   # LiteSVM end to end, incl. the signed head
```

Golden leaves pinned on-chain in the LiteSVM suite: the signed V6 receipt example, the V6
full/none reputation-block vectors, and the three decision-outcome attestation vectors.

## Off-chain half

The publisher lives in the `twzrd-agent-intel` package on PyPI (MIT; `log_anchor_publisher.py`
builds the `[ed25519, anchor_head]` transaction from a signed tree head, and `anchor_log_head.py` is the
CLI that reads the current head from `https://intel.twzrd.xyz/v1/log/sth`, verifies it under the pinned
log key, and anchors only when `tree_size` grew).

## Deployments

| Cluster | Program | Upgrade authority | Notes |
|---|---|---|---|
| mainnet | `BzBAYJxUtJp6mUkJPjEYjd8vdb2FUGnAfB5X9LqrQ72W` | `4HxZL3SAjBcXXk4VJMRs6JSP3T1cW95TkvJY1Ra5GD1B` (BPF upgradeable loader; program data `9e2Q1rDEmvqAFQ9Quh8tvzDAjW8xRxnYTspXvZCdD6CQ`) | last deployed slot 444956973 (2026-09-07 02:56 UTC); anchoring TWZRD's receipt log since 2026-09-07, every 6 hours since 2026-10-01 (only when the log grew). Not yet audited. |
| devnet | `BzBAYJxUtJp6mUkJPjEYjd8vdb2FUGnAfB5X9LqrQ72W` | throwaway devnet key | deployed 2026-09-04, tx `3b3NkCMp995aeui59ocQFMJ9GjAeS9ZZAfxGukeKXeYRcYgBb2yVu9dcMwY8sTFKyAWEaD1JfXmqXpM1zFc5PTX4` |

Check a mainnet anchor yourself: `GET https://intel.twzrd.xyz/v1/log/anchors` lists each anchored head with
its transaction; the transaction invokes `Ed25519SigVerify111111111111111111111111111` and then this program.

First anchored head on devnet (signed by `Ak5SQwHpuQAqU7ty7ZWX7qgF39A9yi72c22KNn8sHzvS`):

- ledger `8EpPmDkQQ9uqMQH166kx7nu4wT415bobWcWfyiZDvgv3` — `log_id intel.twzrd.xyz/v6`, scheme 2, trusted signer = the v2 key
- `anchor_head` tx `5tgSkxJmMpgPLnGJxXAV3eyPvE75J5nVNJdeABLTCnNhzHzrPq9L3n3DRC1dYDoC1BV7tQCEjJEUNyc3pxbvymB4`, slot 492922104
- root entry `yAKKnCafUxus8hT81wXKcs3PZLpyncn1KNEwY2r9Yce` — tree_size 1, root `0x811e1fee65f06c5cfcfee8f338e933c1d3dd261c4c09b8f2793b62bea7ea6db4`, signed timestamp 1788450541

## Cost

Measured on mainnet on 2026-10-01: 0.0014324 SOL per anchored head (1,422,400 lamports of rent for the
permanent 152-byte account plus a 10,000-lamport fee; the first two mainnet anchors, on 09-07 and 09-08, paid
0.00178 SOL, when the rent-exempt minimum was higher). At those early-September rates:
0.001621 SOL once per ledger, 0.288 SOL once for the program. A run is idempotent, so spend tracks the number
of distinct heads, not wall-clock time.

## License

MIT. See [LICENSE](./LICENSE).
