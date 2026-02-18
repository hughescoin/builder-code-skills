---
name: implementing-builder-codes
description: Integrate Base Builder Codes (ERC-8021) into web3 applications for onchain transaction attribution. Use when a project needs to append dataSuffix to transactions on Base L2, whether using Wagmi, Viem, or Privy. Covers EOA transactions, ERC-4337 smart wallets, and EIP-5792 sendCalls. Handles project analysis to detect frameworks, locating transaction call sites, and replacing them with attributed versions.
---

# Builder Codes Integration

Integrate [Base Builder Codes](https://base.dev) into an onchain application. Builder Codes append an ERC-8021 attribution suffix to transaction calldata so Base can attribute activity to your app. No smart contract changes required.

## Prerequisites

- A Builder Code from [base.dev](https://base.dev) > Settings > Builder Codes
- The `ox` library for generating ERC-8021 suffixes: `npm install ox`

## Integration Workflow

Copy this checklist and track progress:

```
Builder Codes Integration:
- [ ] Step 1: Analyze the project (detect framework and wallet type)
- [ ] Step 2: Install dependencies
- [ ] Step 3: Generate the dataSuffix constant
- [ ] Step 4: Apply attribution (framework-specific)
- [ ] Step 5: Verify attribution is working
```

### Step 1: Analyze the project

Determine the framework and wallet setup by inspecting:

```bash
# Check package.json for framework
grep -E "wagmi|@privy-io/react-auth|viem" package.json

# Check for smart wallet / account abstraction usage
grep -rn "useSendCalls\|sendCalls\|ERC-4337\|useSmartWallets" src/

# Check for EOA transaction patterns
grep -rn "useSendTransaction\|sendTransaction\|writeContract\|useWriteContract" src/

# Check Privy version if present
grep "@privy-io/react-auth" package.json
```

**Decision tree:**

- **Privy detected** (`@privy-io/react-auth` v3.13.0+) → See [reference/privy.md](reference/privy.md)
- **Wagmi detected** (without Privy) → See [reference/wagmi.md](reference/wagmi.md)
- **Viem only** (no React framework) → See [reference/viem.md](reference/viem.md)

### Step 2: Install dependencies

```bash
npm install ox
```

Requires `viem >= 2.45.0` for Wagmi/Viem paths. Privy requires `@privy-io/react-auth >= 3.13.0`.

### Step 3: Generate the dataSuffix constant

Create a shared constant (e.g., `src/lib/attribution.ts` or `src/constants/builderCode.ts`):

```typescript
import { Attribution } from "ox/erc8021";

export const DATA_SUFFIX = Attribution.toDataSuffix({
  codes: ["YOUR-BUILDER-CODE"], // Replace with your code from base.dev
});
```

### Step 4: Apply attribution

Follow the framework-specific guide:

- **Privy**: See [reference/privy.md](reference/privy.md) — plugin-based, one config change
- **Wagmi (client-level)**: See [reference/wagmi.md](reference/wagmi.md) — add `dataSuffix` to config
- **Viem (client-level)**: See [reference/viem.md](reference/viem.md) — add `dataSuffix` to wallet client
- **Smart Wallets (EIP-5792 sendCalls)**: See [reference/smart-wallets.md](reference/smart-wallets.md) — pass via `capabilities`

**Preferred approach**: Configure at the **client level** so all transactions are automatically attributed. Only use per-transaction approach if you need conditional attribution.

### Step 5: Verify attribution

1. **base.dev**: Check Onchain > Total Transactions for attribution counts
2. **Block explorer**: Find tx hash, view input data, confirm last 16 bytes are `8021` repeating
3. **Validation tool**: Use [builder-code-checker.vercel.app](https://builder-code-checker.vercel.app/)

## Key Facts

- Builder Codes are ERC-721 NFTs minted on Base
- The suffix is appended to calldata; smart contracts ignore it (no upgrades needed)
- Gas cost is negligible: 16 gas per non-zero byte
- Analytics on base.dev currently support Smart Account (AA) transactions; EOA support is coming (attribution data is preserved)
- The `dataSuffix` plugin in Privy appends to **all chains**, not just Base. If chain-specific behavior is needed, contact Privy
- Privy's `dataSuffix` plugin is NOT yet supported with `@privy-io/wagmi` adapter

## Finding Transaction Call Sites

When retrofitting an existing project, search for these patterns:

```bash
# React hooks (Wagmi)
grep -rn "useSendTransaction\|useSendCalls\|useWriteContract\|useContractWrite" src/

# Viem client calls
grep -rn "sendTransaction\|writeContract\|sendRawTransaction" src/

# Privy embedded wallet calls
grep -rn "sendTransaction\|signTransaction" src/

# ethers.js (if migrating)
grep -rn "signer.sendTransaction\|contract.connect" src/
```

For client-level integration (Wagmi/Viem/Privy), you typically only need to modify the config file — individual transaction call sites remain unchanged.
