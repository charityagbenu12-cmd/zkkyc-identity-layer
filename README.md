# zkKYC Identity Layer

A privacy-preserving identity verification system built on the **Stellar blockchain** using **zero-knowledge proofs**. Users prove their age and identity without ever revealing sensitive personal data — credentials are issued as on-chain Soroban smart contract entries, anchored to the Stellar network.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Stellar Integration](#stellar-integration)
- [Smart Contract](#smart-contract)
- [ZK Proof System](#zk-proof-system)
- [Getting Started](#getting-started)
- [Environment Setup](#environment-setup)
- [Contract Deployment](#contract-deployment)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Stellar Compliance Checklist](#stellar-compliance-checklist)
- [Contributing](#contributing)

---

## Overview

zkKYC Identity Layer allows users to:

- **Prove age** (18+ or 21+) without revealing their actual birthdate
- **Connect** via Freighter wallet on Stellar testnet or mainnet
- **Submit** a ZK proof to a Soroban smart contract for on-chain credential issuance
- **Bind** their credential to a [Human.tech](https://human.tech) Soulbound Token (SBT)
- **Prevent replay attacks** via nullifier-based double-spend protection

No personal data ever leaves the user's device. Only a cryptographic proof hits the chain.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Next.js Frontend                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Connect      │  │ Age          │  │ Credential    │  │
│  │ Wallet       │  │ Verification │  │ Display       │  │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  │
│         │                 │                  │           │
│  ┌──────▼─────────────────▼──────────────────▼───────┐  │
│  │              lib/stellar/client.ts                 │  │
│  │         (Freighter + Soroban RPC)                  │  │
│  └──────────────────────┬─────────────────────────────┘  │
└─────────────────────────┼───────────────────────────────┘
                          │ Soroban RPC
          ┌───────────────▼──────────────────┐
          │     Stellar Network (Testnet)     │
          │  ┌────────────────────────────┐  │
          │  │  zkKYC Verifier Contract   │  │
          │  │  (Soroban / Rust)          │  │
          │  │  - verify_age_proof()      │  │
          │  │  - has_credential()        │  │
          │  │  - bind_to_humantech()     │  │
          │  └────────────────────────────┘  │
          └──────────────────────────────────┘
                          │
          ┌───────────────▼──────────────────┐
          │     Human.tech SBT Contract      │
          │  (Identity Binding Layer)        │
          └──────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14, TypeScript, Tailwind CSS |
| ZK Proofs | Noir (Aztec), UltraHonk backend |
| Blockchain | Stellar (Soroban smart contracts) |
| Wallet | Freighter Browser Extension |
| Contract Language | Rust (`soroban-sdk`) |
| Identity Binding | Human.tech SBTs |

---

## Stellar Integration

This project is fully integrated with the Stellar ecosystem:

### Freighter Wallet

Users connect using [Freighter](https://freighter.app), the official Stellar browser wallet. The integration handles:

- Wallet detection and connection
- Network switching (testnet / mainnet / futurenet)
- Transaction signing and submission

```typescript
// lib/stellar/client.ts
import { connectWallet, isFreighterInstalled } from '@/lib/stellar/client';

const wallet = await connectWallet();
// Returns: { connected: true, address: 'G...', network: 'testnet' }
```

### Soroban Smart Contract

The verifier contract is written in Rust using `soroban-sdk` and deployed to Stellar's smart contract platform. It handles proof verification, credential issuance, and nullifier tracking.

```rust
// contracts/zkkyc_verifier/src/lib.rs
pub fn verify_age_proof(
    env: Env,
    user: Address,
    proof: BytesN<388>,       // UltraHonk proof
    public_inputs: Vec<u128>, // [timestamp, age, credential_hash, nullifier]
) -> bool {
    user.require_auth(); // Stellar auth requirement

    // Replay protection via nullifier
    let nullifier_bytes = Self::u128_to_bytes32(&env, nullifier_u128);
    if nullifiers.contains_key(nullifier_bytes.clone()) {
        return false; // Prevent double-spend
    }

    // Verify ZK proof and issue on-chain credential
    let is_valid = Self::verify_ultra_honk_proof(&env, &proof, &public_inputs);
    if is_valid {
        env.storage().persistent().set(
            &DataKey::Credential(user.clone(), threshold),
            &credential,
        );
        env.events().publish((EVENT_VERIFIED, user), (threshold as u32, credential_hash));
    }
    is_valid
}
```

### Network Configuration

```typescript
// lib/types.ts
export const NETWORK_CONFIG = {
  testnet: {
    horizonUrl: 'https://horizon-testnet.stellar.org',
    sorobanRpcUrl: 'https://soroban-testnet.stellar.org',
    networkPassphrase: 'Test SDF Network ; September 2015',
  },
  mainnet: {
    horizonUrl: 'https://horizon.stellar.org',
    sorobanRpcUrl: 'https://soroban.stellar.org',
    networkPassphrase: 'Public Global Stellar Network ; September 2015',
  },
};
```

### Explorer Links

Transactions are linked to [Stellar Expert](https://stellar.expert) for full on-chain transparency:

```typescript
export function getExplorerUrl(hash: string, network: StellarNetwork): string {
  const baseUrls = {
    testnet: 'https://stellar.expert/explorer/testnet/tx/',
    mainnet: 'https://stellar.expert/explorer/public/tx/',
  };
  return baseUrls[network] + hash;
}
```

---

## Smart Contract

Located in `contracts/zkkyc_verifier/src/lib.rs`.

### Key Functions

| Function | Description |
|---|---|
| `initialize(admin, vk)` | Deploy and configure the contract |
| `verify_age_proof(user, proof, inputs)` | Verify ZK proof and issue credential |
| `has_credential(user, threshold)` | Check if user holds a valid credential |
| `get_credential(user, threshold)` | Fetch full credential details |
| `bind_to_humantech(user, threshold, sbt_id)` | Link credential to Human.tech SBT |
| `revoke_credential(user, threshold)` | Admin: revoke a credential |
| `pause() / unpause()` | Admin: emergency stop |

### Credential Storage

Credentials are stored in **persistent** Soroban storage, keyed by `(Address, AgeThreshold)`:

```rust
#[contracttype]
pub struct VerifiedCredential {
    pub user: Address,
    pub threshold: AgeThreshold,   // Over18 | Over21
    pub credential_hash: BytesN<32>,
    pub issued_at: u64,
    pub expires_at: u64,
    pub revoked: bool,
    pub humantech_sbt_id: u128,    // 0 = not bound
}
```

---

## ZK Proof System

ZK circuits are written in [Noir](https://noir-lang.org) and located in `circuits/age_verification/`.

The circuit proves:
- The user's birthdate is before `current_timestamp - required_age * SECONDS_PER_YEAR`
- Without revealing the actual birthdate

```noir
// circuits/age_verification/src/main.nr (simplified)
fn main(
    birthdate_timestamp: Field,   // private
    credential_secret: Field,     // private
    current_timestamp: pub Field, // public
    required_age: pub Field,      // public
    credential_hash: pub Field,   // public
    nullifier: pub Field          // public
) {
    // Prove age without revealing birthdate
    let age_in_seconds = current_timestamp - birthdate_timestamp;
    let required_seconds = required_age * SECONDS_PER_YEAR;
    assert(age_in_seconds >= required_seconds);
}
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- [Freighter wallet](https://freighter.app) browser extension
- Rust + `soroban-cli` (for contract deployment)

### Install & Run

```bash
# Clone the repo
git clone https://github.com/your-org/zkkyc-identity-layer.git
cd zkkyc-identity-layer

# Install dependencies
npm install

# Start the dev server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

---

## Environment Setup

Create a `.env.local` file in the project root:

```env
NEXT_PUBLIC_STELLAR_NETWORK=testnet
NEXT_PUBLIC_VERIFIER_CONTRACT_ADDRESS=<your_deployed_contract_address>
NEXT_PUBLIC_HUMANTECH_CONTRACT_ADDRESS=CCNTHEVSWNDOQAMXXHFOLQIXWUINUPTJIM6AXFSKODNVXWA4N7XV3AI5
NEXT_PUBLIC_SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
```

---

## Contract Deployment

### 1. Install Soroban CLI

```bash
cargo install --locked soroban-cli
```

### 2. Build the Contract

```bash
cd contracts/zkkyc_verifier
cargo build --target wasm32-unknown-unknown --release
```

### 3. Deploy to Testnet

```bash
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/zkkyc_verifier.wasm \
  --source <your_secret_key> \
  --network testnet
```

### 4. Initialize the Contract

```bash
soroban contract invoke \
  --id <contract_address> \
  --source <your_secret_key> \
  --network testnet \
  -- initialize \
  --admin <admin_stellar_address> \
  --verification_key <768_byte_vk_hex>
```

### 5. Update Contract Address

Add the deployed contract address to your `.env.local` and to `lib/types.ts` under `CONTRACT_ADDRESSES.testnet.verifier`.

---

## Usage

1. **Connect Wallet** — Click "Connect Wallet" and approve in Freighter
2. **Enter Birthdate** — Input your birthdate (stays private, never sent anywhere)
3. **Generate Proof** — The Noir circuit generates a ZK proof in-browser
4. **Submit to Stellar** — The proof is submitted to the Soroban verifier contract
5. **Credential Issued** — An on-chain credential is stored under your Stellar address
6. **Optional: Bind to Human.tech** — Link your credential to a Human.tech SBT for cross-platform identity

---

## Project Structure

```
zkkyc-identity-layer/
├── app/                        # Next.js app router
├── components/
│   └── zkkyc/
│       ├── connect-wallet.tsx      # Freighter wallet connection
│       ├── age-verification-form.tsx
│       ├── proof-generator.tsx     # Noir proof generation
│       ├── verification-status.tsx
│       └── credential-display.tsx
├── lib/
│   ├── stellar/
│   │   └── client.ts           # Stellar/Soroban client
│   ├── noir/
│   │   ├── circuits.ts         # Circuit management
│   │   └── prover.ts           # Proof generation
│   └── types.ts                # Shared TypeScript types
├── circuits/
│   └── age_verification/
│       └── src/main.nr         # Noir ZK circuit
├── contracts/
│   └── zkkyc_verifier/
│       └── src/
│           ├── lib.rs          # Soroban verifier contract
│           └── humantech.rs    # Human.tech integration
└── README.md
```

---

## Stellar Compliance Checklist

- [x] Uses **Freighter** as the official Stellar wallet integration
- [x] Supports **testnet**, **mainnet**, and **futurenet** network switching
- [x] Contract built with **`soroban-sdk`** (official Stellar smart contract SDK)
- [x] Uses **`user.require_auth()`** for all user-initiated operations
- [x] Uses **persistent storage** for long-lived credential data
- [x] Uses **instance storage** for contract-level config and counters
- [x] Emits **Soroban events** on credential issuance, revocation, and binding
- [x] Correct **network passphrases** for testnet and mainnet
- [x] Integrates with **Horizon** and **Soroban RPC** endpoints
- [x] Transaction links point to **Stellar Expert** explorer
- [x] Validates **Stellar address format** (`G` prefix, 56 chars)
- [x] Implements **nullifier-based replay protection**
- [x] Admin functions protected by **`require_auth()`**
- [x] Contract supports **emergency pause** mechanism
- [x] Human.tech SBT contract address matches **mainnet deployment**

---

## Contributing

Pull requests are welcome. For major changes, open an issue first to discuss what you'd like to change.

---


