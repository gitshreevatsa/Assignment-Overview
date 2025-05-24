# Debug Report: `eth_getBlockByNumber` Error on Sei QuickNode

## Problem Statement

A team encountered the following error while querying a specific block on Sei’s EVM-compatible RPC via QuickNode:

```bash
curl --request POST \
  --url https://sleek-icy-sanctuary.sei-pacific.quiknode.pro/<redacted> \
  --header 'Content-Type: application/json' \
  --header 'x-qn-height: 78309999' \
  --data '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["0x4AAEA6F",true], "id":1}'
```

Response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "could not find block for height 826401374464"
  }
}
```

## Investigation Approach

To understand and reproduce the issue, I followed a step-by-step debugging process:

---

### Step 1: **Reproduced the issue**

I ran the same request using `curl` with the same block number and header and confirmed I got the same error.

---

### Step 2: **Isolate header effects**

I removed the `x-qn-height` header to check if it had any effect.
→ **Same error** — header didn’t seem to influence the issue here.

---

### Step 3: **Cross-provider comparison**

Queried the same block using **Alchemy** instead of QuickNode:

* On Alchemy: ✅ `eth_getBlockByNumber("0x4AAEA6F", true)` **works fine**
* On QuickNode: ❌ Same request still fails

---

### Step 4: **Test surrounding block numbers**

I started testing nearby block numbers on QuickNode:

* `0x4AAEA6F` (the original): ❌ fails
* `0x4AAEA70` (blockNumber + 1): ❌ fails
* `0x4AAEA71` (blockNumber + 2): ✅ **works**

So the earliest retrievable block on QuickNode is `blockNumber + 2`.

---

### Step 5: **Verified with block hashes**

Using the Sei explorer, I fetched the block hashes:

* Queried `eth_getBlockByHash` for the hash of block `0x4AAEA6F`: ❌ "block not found"
* Queried for the hash of `0x4AAEA71`: ✅ works fine

---

### Step 6: **Checked for historical data availability**

I went back about a year in block history and found that older blocks have been **pruned** — they are no longer available via QuickNode. This strongly suggests QuickNode is only maintaining a recent subset of EVM block data.

---

## Root Cause Summary

QuickNode appears to **prune older EVM blocks** on the Sei chain and **only retains data from a recent base height onwards**. That base height is advertised in the error messages or may correspond to internal QuickNode data retention settings.

In this case:

* The actual error blockNumber (`0x4AAEA6F` = `78209999`) is **below** the retained base height.
* The earliest available block is `base height + 2`, confirmed via both block number and hash. This was at the time of investigation and may change when tested again as time progresses, but the pattern of pruning is clear.

## User Guidance

If a user queries a pruned block on QuickNode:

1. **Use an alternative RPC provider like Alchemy**, which retains older block data.
2. **Avoid relying on QuickNode for full historical Sei data.** It's best suited for recent blocks and real-time data access.
3. **For archival needs**, recommend teams to either:

   * Run their own Sei full archival node
   * Use a provider known to store full EVM history

---

## Optional Fixes

You could implement a simple middleware API to patch this:

* Detect if block number is below QuickNode's base height
* Redirect the query to a fallback RPC (like Alchemy)
* Return a unified response to the caller

This helps ensure your internal systems or user tools continue to work without breaking due to provider limitations.

---

## How I'd Respond to the Asking Team

> “The block you’re trying to query is no longer available via QuickNode as it has likely pruned historical data below its base height. I validated this by querying several nearby blocks and confirmed that only blocks from base height + 2 are available(at the time of testing and quicknode prunes blocks that are of an year old). You can either use another RPC provider like Alchemy, which retains older blocks, or consider running an archival node if historical access is a requirement for your use case.”

Additionally, I would reach out to the QuickNode team to highlight two things:

- Their RPC advertises a base height, but blocks at or above this height can still return pruning errors the actual minimum available height should be clarified or exposed accurately.

- When a block is unavailable, the error message returns height like 826401374464, which doesn't map to any real EVM or Cosmos block height. This could confuse developers and mislead automated tooling. Suggesting they replace this with a clearer error (e.g., "block height pruned" or "block unavailable due to retention policy") would improve developer experience.

## Closing Thoughts

Through this debugging process, I validated the block availability using both `eth_getBlockByNumber` and `eth_getBlockByHash`, confirmed the behavior across multiple RPCs, and triangulated against explorer data. I didn’t assume the RPC response at face value; instead, I tested boundaries, alternate paths, and infrastructure behaviors — which helped isolate that the issue lies not with the method or parameters, but with QuickNode’s data retention model.


## AI Assistance Disclosure
I used ChatGPT 4 Turbo to help draft the readme after providing it all my manual steps and my inferences.