# Solo Precompile: Selectively Claiming CW20 and CW721 Assets from Cosmos Accounts

The Solo precompile is a built-in contract on Sei that enables users to migrate tokens from Cosmos accounts to EVM accounts by making a single smart contract call. With the latest update, users can now claim **specific** CW20 tokens or CW721 NFTs (not just all tokens) using a new, more flexible method. This streamlines asset migration for dApps and users who want granular control over which assets to move.

> **What is a precompile?**  
> A precompile is a special smart contract, deployed at a fixed address by the Sei protocol itself, that exposes native chain logic to EVM-based applications. It acts like a regular EVM contract, but can access privileged, low-level features such as Cosmos asset transfers or signature validation.

---

## What Is Introduced in This Update?

- **Selective Asset Claims:**  
  Users can now migrate *only* chosen CW20 tokens or CW721 NFTs (or both) from a Cosmos account to an EVM address, instead of being forced to claim all tokens.
- **New Function:**  
  The Solo precompile now exposes `claimSpecific(bytes payload)`, which takes a signed payload that specifies exactly which tokens/NFTs to claim.
- **CLI Support:**  
  The `seid` CLI provides an easy way to generate the required payload for any combination of CW20 and CW721 assets.

---

## Functions

The Solo precompile exposes these functions:

```solidity
/**
 * @dev Claim all assets using approver's signed Cosmos tx payload.
 * @param payload Signed Cosmos tx payload as bytes.
 * @return response true indicates a successful claim.
 */
function claim(bytes memory payload) external returns (bool response);

/**
 * @dev Claim specific assets (CW20/CW721) using signed Cosmos tx payload.
 * @param payload Signed Cosmos tx payload as bytes (specifying assets).
 * @return response true indicates a successful claim.
 */
function claimSpecific(bytes memory payload) external returns (bool response);
```

- **claim(bytes payload):** Accepts a signed Cosmos transaction payload and, if valid, transfers all assets from the Cosmos account (sender) to the EVM account (caller).
- **claimSpecific(bytes payload):** Accepts a signed Cosmos transaction payload listing the assets to claim. If valid, transfers only those tokens/NFTs from the Cosmos account (sender) to the EVM account (caller).

---

## How Does Selective Asset Claiming Work?

The Solo precompile at address `0x000000000000000000000000000000000000100C` now exposes two functions:
- `claim(bytes payload)`: Claims *all* tokens from a Cosmos account.
- `claimSpecific(bytes payload)`: Claims only specific CW20/CW721 assets you choose.

**How the process works:**

1. **Payload Generation:**  
   Use the Sei CLI to generate a signed claim payload specifying which assets to claim.
2. **Contract Call:**  
   Submit this payload to `claimSpecific` via a contract call from your EVM account.
3. **Asset Migration:**  
   The precompile verifies the payload and migrates only the selected tokens/NFTs from the Cosmos address to your EVM address.

---

## Use Cases

- **Selective Token Migration:**  
  Move only certain CW20 tokens (e.g., stablecoins) or specific NFTs, leaving the rest untouched.
- **Granular User Onboarding:**  
  Let users bring over just the assets they want from Cosmos to EVM.
- **Partial Account Migration:**  
  For developers/dApps that want users to claim only app-specific tokens or collections.

---

## Step-by-Step Guide: Selectively Claiming CW20 and CW721 Assets

### Prerequisites

- Sei node RPC (devnet or mainnet)
- Node.js and Hardhat installed
- `seid` CLI available
- Access to the Sei Chain repo with this feature


### 1. Create a Cosmos Account

To interact with Cosmos-based tokens or NFTs, you first need a Cosmos account on Sei. This is different from your EVM (MetaMask) address.

**Creating a Cosmos Account:**

```sh
seid keys add solo
```
This creates a new Cosmos account named `solo`. You can view its address with:

```sh
seid keys show solo -a
```

---

### 2. Fund Your Cosmos Account (Testnet Only)

If you are on testnet, your Cosmos account will need tokens to pay for transactions.  
You can use the official Sei faucet to get testnet tokens:

- [Sei Testnet Faucet](https://www.docs.sei.io/learn/faucet)

Follow the instructions there to claim tokens for your Cosmos address.

---

### 3. Get Your EVM Account Address

Generate or use an existing EVM address (for example, from Hardhat, MetaMask, or your preferred wallet).  
This is where your tokens or NFTs will be claimed to.

---

### 4. Generate the Claim-Specific Payload

Use the CLI to specify which assets to claim. The syntax is:

```sh
seid tx evm print-claim-specific <evm_address> [CW20 <cw20_contract>] [CW721 <cw721_contract>] ... --from solo -y
```

- Replace `<evm_address>` with your EVM account address.
- For each asset, add a type (`CW20` or `CW721`) and the contract address.
- Example: To claim one CW20 token and one NFT collection:

```sh
seid tx evm print-claim-specific 0xYourEVMAddress CW20 sei1cw20contract CW721 sei1cw721contract --from solo -y
```

- Save the hex-encoded output for the next step.

---

### 5. Call the Solo Precompile from JavaScript

Below is a minimal example using Ethers.js to perform the claim. You could run this as Hardhat tasks or a script to interact with the Solo precompile and claim your assets.
**Note:** The payload output by the CLI is a hex string—**you must convert it to bytes** before passing to the contract call. Use the updated `hex2uint8` helper below.

```javascript
const { ethers } = require("hardhat");
const abi = require("../../precompiles/solo/abi.json");
const soloPrecompile = "0x000000000000000000000000000000000000100C";

// Helper: Convert a hex string to Uint8Array (required for payload argument)
function hex2uint8(hex) {
    const hex_chars = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'];
    hex = hex.toUpperCase();
    let uint8 = new Uint8Array(Math.floor(hex.length/2));
    for (let i=0; i < Math.floor(hex.length/2); i++) {
      uint8[i] = hex_chars.indexOf(hex[i*2])*16;
      uint8[i] += hex_chars.indexOf(hex[i*2+1]);
    }
    return uint8;
}

async function main() {
  const [signer] = await ethers.getSigners();

  // Use the payload generated from the CLI
  const payloadHex = "0x..."; // Paste your payload here
  const payload = hex2uint8(payloadHex.startsWith('0x') ? payloadHex.slice(2) : payloadHex);

  const solo = new ethers.Contract(soloPrecompile, abi, signer);

  // Call the claimSpecific function
  const tx = await solo.claimSpecific(payload, { gasLimit: 200000 });
  const receipt = await tx.wait();

  if (receipt.status === 1) {
    console.log("Claim successful!");
  } else {
    console.log("Claim failed.");
  }
}

main().catch(console.error);
```

---

## How to Check That the Claim Worked

After running your claim, you can check your EVM account in your wallet (such as MetaMask):

- **CW20 tokens:**  
  Add the CW20 token contract address as a custom token in MetaMask. The new balance should appear under your EVM address.
- **CW721 NFTs:**  
  Use a block explorer or a dApp that displays NFTs for your EVM address. The claimed NFTs should now be owned by your EVM address.

If your wallet or dApp does not support these custom assets, you can also use EVM contract calls (such as `balanceOf` for CW20, or `ownerOf` for CW721) to verify ownership.

### How to Find the EVM Address for a CW20/CW721 Contract

If you have a Cosmos Bech32 contract address (e.g. `sei1...`), you need the EVM (`0x...`) representation to add it to MetaMask or use in EVM tools.

**To convert the native asset address to EVM hex address:**

```sh
seid debug addr <native_asset_address>
```
- Replace `<native_asset_address>` with your CW20/CW721 contract's Sei (Bech32) address.
- The output will include the corresponding EVM hex address (starting without `0x`).
- This is the address you should use as the "Token Contract Address" in MetaMask or other EVM wallets.

---

## Troubleshooting

- **Gas issues:** Set gas limit to at least 200,000 (increase if migrating many NFTs).
- **Invalid payload:** Always use the output from `seid tx evm print-claim-specific ...`.
- **Sequence mismatch:** The Cosmos account must not make transactions after generating the payload.
- **Ownership not updated:** If tokens/NFTs are not received, double-check addresses, asset types, and that the correct contracts were specified.

---

## Reference

- [Full E2E JavaScript Test Example (`SeiSoloTest.js`)](https://github.com/sei-protocol/sei-chain/blob/main/contracts/test/SeiSoloTest.js)
- [Pull Request: Add ClaimSpecific for CW20/CW721](https://github.com/sei-protocol/sei-chain/pull/2138)
- [Sei Docs](https://docs.sei.io/)
- [Sei Testnet Faucet](https://www.docs.sei.io/learn/faucet)

---
## Thinking Process

This guide focuses on [PR #2138](https://github.com/sei-protocol/sei-chain/pull/2138), which extends the Solo precompile with `claimSpecific()` to allow granular migration of CW20 and CW721 assets. Since the feature had not yet been merged or deployed to a live network, and running a custom Sei node locally requires high system resources, I approached the task by analyzing the PR code and CLI tool usage examples.

The goal was to create a practical guide that walks through the expected developer workflow — from Cosmos asset selection to EVM-based claim — even if the function could not be tested directly. I aimed to cover not just code interaction but also real-world use cases like partial onboarding and selective migration.

## Limitations & Context

This documentation is based on features introduced in [PR #2138](https://github.com/sei-protocol/sei-chain/pull/2138), which were still open pull requests at the time of writing. These precompiles are not deployed on Sei devnet or mainnet, and as precompiles, they require node-level integration rather than smart contract deployment.

Due to the lack of public availability and the high hardware requirements for compiling and running a custom Sei node locally, the tutorials have not been executed end-to-end. Instead, they are based on code-level understanding, CLI reference, and expected usage patterns. The guides are written to be fully testable once the features are live.

  
## AI Assistance Disclosure
I used AI, ChatGPT 4 Turbo to assist me in drafting the docs in a more structured way after putting in my raw docs data learning from the PR. It helped me generate the reference links and helped me fetch the seid command for generating the evm address from the native asset address. 