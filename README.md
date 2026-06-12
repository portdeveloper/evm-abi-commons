# evm-abi-commons

An open, **provably-correct** EVM selector → signature dataset. Like
4byte.directory, but every entry carries a proof grade — and it covers the
contracts nobody verified.

**License: CC0-1.0** (public domain). Use it in anything, no attribution needed.

## Why another signature database?

Existing selector databases accept open submissions, which is how they got
poisoned with collision-mined junk (a 4-byte selector is only 32 bits — mining a
plausible-looking wrong signature is cheap). This dataset takes the opposite
approach: **no submissions**. Every entry is produced by one resolution
pipeline ([gulltoppr](https://github.com/portdeveloper/gulltoppr)) and graded by
how it was proven:

| `proof` | Meaning |
|---|---|
| `verified-source` | Extracted from a contract whose source is verified on-chain (Etherscan/Sourcify). Ground truth — name, types, outputs, mutability, param names all come from real source. |
| `keccak-proven` | Proposed by an LLM for a selector in *unverified* (decompiled) bytecode, accepted **only** when `keccak256(signature)[:4]` reproduces the on-chain selector. Name + param types are proven; semantics, outputs and mutability are not. |

Notes on trust:
- **Events are airtight**: event entries are keyed by the full 32-byte `topic0`
  keccak hash — zero collision risk, cryptographically solid.
- A proven *signature* is not a proven *behavior*: `claimReward(address)`
  matching a selector does not prove the function claims a reward. Provenance
  tells you what was proven; treat semantics as inferred.
- Errors use 4-byte selectors like functions.

## Data

JSON Lines, one entry per line, deterministic ordering:

| file | entries |
|---|---|
| `data/functions.jsonl` | 3,337 |
| `data/events.jsonl` | 905 |
| `data/errors.jsonl` | 1,268 |
| `data/all.jsonl` | 5,510 (all of the above) |

### Schema

```json
{
  "selector": "0xa9059cbb",            // 4-byte (function/error) or 32-byte topic0 (event)
  "kind": "function",                  // "function" | "event" | "error"
  "signature": "transfer(address,uint256)",
  "proof": "verified-source",          // "verified-source" | "keccak-proven"
  "abi_item": { ... },                 // full ABI item incl. param names/outputs (when proof is verified-source)
  "chain": 1,                          // first-sighting chain id (informational)
  "address": "0x..."                   // first-sighting address (informational)
}
```

## Live API

The dataset is a snapshot of gulltoppr's live registry, which grows as a
byproduct of resolution (every verified contract the engine resolves is
harvested; every decompile can mint keccak-proven names):

```bash
curl https://api.gulltoppr.dev/v1/lookup/0xa9059cbb        # one selector (or 32-byte topic0)
curl https://api.gulltoppr.dev/v1/registry/stats           # current counts
curl https://api.gulltoppr.dev/v1/registry/export          # full JSONL dump (this dataset)
```

## Regenerating

```bash
curl -s https://api.gulltoppr.dev/v1/registry/export -o data/all.jsonl
# split by kind however you like; this repo splits into functions/events/errors
```

## Provenance of the pipeline

Produced by [🐴 gulltoppr](https://github.com/portdeveloper/gulltoppr) — the
agent-native interface to any EVM contract (resolve ABIs even for unverified
contracts via decompilation, read, simulate, prepare non-custodial txs; REST +
MCP + SDK). The registry design (skeleton-hash bytecode keying, verified-corpus
harvesting, type-constrained LLM propose-and-verify) is documented in the main
repo.
