
# Debug Report: Fixing `transactionIndex` Mismatch on Sei EVM

## Issue Overview

Sei’s EVM-compatible RPC exposes an inconsistency between the `eth_getLogs` and `eth_getBlockByNumber` RPC endpoints.

- `eth_getLogs` returns logs with a `transactionIndex` based on the **full block's transaction list**, which includes both **native (Cosmos-style)** and **EVM transactions**.
- However, `eth_getBlockByNumber` only returns the **EVM transactions**, indexed from `0...N-1`.

This leads to broken assumptions in indexers like `ponder.sh` and The Graph, which expect:
```ts
block.transactions[log.transactionIndex] === log.transactionHash
```

---

## Root Cause

- Sei internally processes **both native and EVM txs**.
- EVM indexers only receive the filtered list of EVM txs via `eth_getBlockByNumber`.
- The `transactionIndex` in logs still refers to the original position of the tx in the **full, mixed tx list** (e.g. index 27), even though the block might only return 5 EVM txs (indexed 0–4).

---

## Solution (Patch at Indexer Layer)

My approach:
- Use `eth_getBlockByNumber` to fetch the list of **EVM-only txs**
- Build a map of `txHash → correctedIndex` based on their order in that list
- Fetch logs using `eth_getLogs`
- For each log:
  - Keep the original `transactionIndex` (e.g., `27`)
  - Add a `correctedTransactionIndex` (e.g., `3`) that matches its actual index in the EVM tx list

This allows tools to safely index logs using the corrected value.

---

## Proof of Concept Repo

🔗 [This Repository](https://github.com/gitshreevatsa/ethLog-tx-mismatch) 

---

## Supporting Evidence

The approach mirrors the fix merged in the official Sei repository:
- ✅ [Sei PR #2157 – Assigning correct TxIndex in logs](https://github.com/sei-protocol/sei-chain/pull/2157)

```go
for _, log := range logs {
    log.TxIndex = uint(txCount) // where txCount only includes valid EVM txs
}
```

This validates my logic of assigning transaction index based solely on filtered EVM tx ordering.

---

## Summary

| Problem                                | Fix                                                    |
|----------------------------------------|---------------------------------------------------------|
| Logs use `transactionIndex` from full tx list (native + EVM) | Recalculate index based on EVM-only list from the block |
| Block exposes only EVM txs with reindexed 0..N-1            | Match logs to tx via `transactionHash → index`          |
| Indexers break due to mismatch                             | Patch logs to include `correctedTransactionIndex`       |

This ensures compatibility with EVM indexers and makes Sei logs reliably indexable.

---


## AI Assistance Disclosure
I used AI to assist me in drafting the readme, refactoring my codebase of the repo mentioned here and to write a good readme for that as well. I used Claude for refactoring, Copilot for code completions, and ChatGPT 4 Turbo to help draft the readme and structure the report of my findings and investigations.