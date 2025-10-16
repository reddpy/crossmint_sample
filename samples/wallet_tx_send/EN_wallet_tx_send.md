# Crossmint Wallet Integration Guide

Quick start guide for creating wallets and sending transactions with Crossmint SDK.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage Examples](#usage-examples)
  - [Creating a Wallet](#creating-a-wallet)
  - [Sending a Transaction](#sending-a-transaction)
  - [Complete Flow](#complete-flow)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

## Prerequisites

Before you begin, ensure you have:

- Node.js 16+ installed
- A Crossmint account with API access
- API key with the following scopes:
  - `wallets.create`
  - `wallets:transactions.create`

**Note:** For EVM-compatible chains (e.g., polygon), wallets share an address space derived from the same private key, but must be instantiated per chain for SDK interactions to route correctly. Smart Wallets are supported on many chains (see below).

## Installation
```bash
npm install @crossmint/server-sdk
```
For ERC-20 encoding in examples (optional but recommended for token transfers):
```bash
npm install ethers
```

## Quick Start
```typescript
import { createCrossmint, CrossmintWallets, EVMWallet } from "@crossmint/server-sdk";

const crossmint = createCrossmint({
    apiKey: "your-server-api-key-here",
    // environment: "staging" // Uncomment for staging (limits to testnets)
});

const crossmintWallets = CrossmintWallets.from(crossmint);
```

## Usage Examples

### Creating a Wallet
```typescript
async function createUserWallet(userEmail: string, chain: string = "polygon") {
    try {
        const wallet = await crossmintWallets.createWallet({
            chain,  // See supported chains; EVM chains share keys but need per-chain init
            signer: {
                type: "email",
                email: userEmail
            },
            // owner: `email:${userEmail}` // Optional: Tie to user identity for Crossmint Auth
        });

        console.log("✅ Wallet created successfully!");
        console.log("Address:", wallet.address);
        return wallet;
    } catch (error) {
        console.error("❌ Error creating wallet:", error);
        throw error;
    }
}
```

**Example Usage:**
```typescript
const wallet = await createUserWallet("user@globosend.com", "polygon");
```

### Sending a Transaction
```typescript
import { ethers } from "ethers"; // For encoding ERC-20 data

async function sendStablecoinTransfer(
    senderEmail: string,
    recipientAddress: string,
    amount: string, // e.g., "10.00" (human-readable)
    chain: string = "polygon"
) {
    try {
        // Get the wallet using the email locator (format: email:email@domain:chain)
        const walletLocator = `email:${senderEmail}:${chain}`;
        const wallet = await crossmintWallets.getWallet(walletLocator);

        // Wrap for EVM operations (SDK auto-handles signer approvals, e.g., email magic link)
        const evmWallet = EVMWallet.from(wallet);

        // For native transfer: Set value to hex wei (e.g., ethers.parseEther(amount)), data: "0x", to: recipient
        // For ERC-20 (e.g., USDC on Polygon):
        const USDC_ADDRESS = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"; // Polygon USDC
        const ERC20_ABI = ["function transfer(address to, uint256 amount)"];
        const iface = new ethers.Interface(ERC20_ABI);
        const amountWei = ethers.parseUnits(amount, 6); // USDC: 6 decimals
        const data = iface.encodeFunctionData("transfer", [recipientAddress, amountWei]);

        // Send transaction
        const { hash, explorerLink } = await evmWallet.sendTransaction({
            to: USDC_ADDRESS,     // Contract for tokens; recipient for native
            value: "0x0",         // Hex wei for native; "0x0" for tokens
            data                  // "0x" for simple/native sends
        });

        console.log("✅ Transaction sent!");
        console.log("Hash:", hash);
        console.log("Explorer:", explorerLink);

        return { hash, explorerLink };
    } catch (error: any) {
        console.error("❌ Error sending transaction:", error);
        if (error.message.includes("approval")) {
            console.log("Check email for magic link approval.");
        }
        throw error;
    }
}
```

**Example Usage:**
```typescript
const result = await sendStablecoinTransfer(
    "sender@globosend.com",
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "10.00"  // $10 USDC
);
```

### Complete Flow
```typescript
async function main() {
    // Step 1: Create wallet for sender
    const senderEmail = "sender@globosend.com";
    const chain = "polygon";
    const wallet = await createUserWallet(senderEmail, chain);

    // Step 2: Send transfer to recipient (ensure wallet is funded for fees/gas)
    const recipientAddress = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
    const result = await sendStablecoinTransfer(
        senderEmail,
        recipientAddress,
        "10.00",  // $10 USDC
        chain
    );

    console.log("✅ Transfer complete:", result);
}

main().catch(console.error);
```

## Configuration

### API Key Setup

1. Go to [Crossmint Console](https://www.crossmint.com/console)
2. Navigate to **Settings** → **API Keys**
3. Create a new server-side API key (never expose on client-side)
4. Ensure it has the required scopes:
   - `wallets.create`
   - `wallets:transactions.create`

**Staging vs Production:** Use `environment: "staging"` in `createCrossmint` for testing (base URL: staging.crossmint.com). Switch to production for mainnet.

### Wallet Locator Format

When retrieving an existing wallet, use flexible identifiers:
```text
email:{user-email}:{chain-type}  // Chain-specific
// Or: email:{user-email}:evm     // For any EVM chain
// Or: {owner-type}:{value}:{chain} (e.g., userId:123:polygon)
```

**Examples:**

- `email:user@example.com:polygon`
- `email:user@example.com:base`
- `email:user@example.com:ethereum-sepolia`
- Direct address: `0x...` (if known)

### Supported Chains

Crossmint supports Smart Wallets on 50+ chains. Below is a subset relevant to EVM wallets (full list includes non-EVM like Solana). ✅ means supported for Smart Wallets; ✱ means available on request (contact sales).

| Blockchain     | Identifier       | Mainnet Label | Testnet Label       | Smart Wallets | Best For          | Fees    | Notes                  |
|----------------|------------------|---------------|---------------------|---------------|-------------------|---------|------------------------|
| Polygon       | polygon         | `polygon`    | `polygon-amoy`     | ✅            | Stablecoins      | Low    | Recommended for transfers |
| Base          | base            | `base`       | `base-sepolia`     | ✅            | Coinbase users   | Low    | Mainnet supported      |
| Ethereum Sepolia | ethereum-sepolia | -           | `ethereum-sepolia` | ✅            | Testing          | Testnet| Staging default        |
| Arbitrum One  | arbitrum        | `arbitrum`   | `arbitrum-sepolia` | ✅            | Layer 2          | Low    |                        |
| Optimism      | optimism        | `optimism`   | `optimism-sepolia` | ✅            | Layer 2          | Low    |                        |
| Solana        | solana          | `solana`     | `solana`           | ✅            | High throughput  | Variable| Non-EVM example       |
| ... (50+ more)| Varies          | Varies       | Varies             | Varies        | Varies           | Varies | See full table below  |

> **Note:** On staging, only testnet chains are supported (e.g., polygon-amoy, not polygon mainnet). For ✱ chains or custom, [contact sales](https://www.crossmint.com/contact/sales). EVM chains share keys.

[Full Supported Chains Table →](https://docs.crossmint.com/introduction/supported-chains) (includes MPC Wallets, Minting, etc.)

### Transaction Parameters

As per EVMTransactionInput:

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `to` | string | Recipient or contract address | `"0x742d35..."` |
| `value` | hex string | Native currency amount in wei (hex) | `"0x0"` (0) or `"0xde0b6b3a7640000"` (1 unit) |
| `data` | hex string | Encoded calldata for contracts | `"0x"` (none) or ERC-20 encoded transfer |

**Signer Approvals:** SDK auto-handles (e.g., email magic link). Expect approval email for transactions.

## Troubleshooting

### Common Errors

#### ❌ `walletLocator is not defined` or Invalid Locator
**Solution:** Use correct format (case-sensitive). Examples:
```typescript
const walletLocator = `email:${userEmail}:polygon`; // Or :evm for EVM
```

#### ❌ `Invalid API key` or Scope Errors
**Solutions:**
- Use server-side API key only
- Verify scopes in Console (`wallets.create`, `wallets:transactions.create`)
- Check staging/prod key mismatch

#### ❌ `Chain not supported`
**Solutions:**
- Exact identifier from table (e.g., "polygon", not custom)
- Testnets on staging only (e.g., "polygon-amoy")
- Check full [supported chains](https://docs.crossmint.com/introduction/supported-chains); contact for ✱

#### ❌ `Wallet already exists`
**Solution:** Use `getWallet()` to retrieve; createWallet will error if duplicate

#### ❌ Transaction Pending or Approval Failed
**Solution:**
- Check spam for email signer magic link
- Ensure wallet has funds for gas/fees
- Poll tx status via API if needed

### Debug Mode

Enable detailed logging:
```typescript
const crossmint = createCrossmint({
    apiKey: "your-api-key",
    debug: true,  // Enable debug logs
});
```

## Resources

- 📖 [Full Documentation](https://docs.crossmint.com/wallets)
- 🔧 [API Reference](https://docs.crossmint.com/api-reference)
- 💬 [Discord Community](https://discord.gg/crossmint)
- 📧 [Support](mailto:support@crossmint.com)
- [Signers & Custody](/wallets/signers-and-custody) for advanced auth
- [Supported Chains Full Table](https://docs.crossmint.com/introduction/supported-chains)

## Next Steps

- [ ] Add proper error handling for production (e.g., retry on approval pending)
- [ ] Implement retry logic for failed transactions
- [ ] Add transaction status polling (via API)
- [ ] Store wallet addresses/locators in your database
- [ ] Set up webhook listeners for transaction events
- [ ] Integrate Crossmint Auth for seamless owner resolution
- [ ] Test on staging with testnets (e.g., polygon-amoy)

## License

This example code is provided as-is for integration purposes.

---

**Need help?** Open an issue or contact support@crossmint.com
