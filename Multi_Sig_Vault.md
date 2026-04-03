# 🔐 Multisig Vault API — The Complete Deep Dive

> *An interview-style, in-depth walkthrough of every major component in the Multisig Vault project.*
> *Think of this as sitting down with the architect of the system and asking them to explain every decision, every line of defense, and every moving part.*

---

## Table of Contents

1. [The Big Picture](#the-big-picture)
2. [Project Setup & Configuration](#project-setup--configuration)
3. [The Type System — Data Modeling](#the-type-system--data-modeling)
4. [Constants — The Anchor Points](#constants--the-anchor-points)
5. [The Store — In-Memory State Management](#the-store--in-memory-state-management)
6. [Utilities — The Engine Room](#utilities--the-engine-room)
7. [Routes — The API Surface](#routes--the-api-surface)
8. [The Server — Tying It All Together](#the-server--tying-it-all-together)
9. [Security Model](#security-model)
10. [How a Full Workflow Plays Out](#how-a-full-workflow-plays-out)

---

## The Big Picture

**Q: In one paragraph, what is this project?**

**A:** This is a **Multisig (multi-signature) Vault API** — a REST server built with Node.js, Express, and TypeScript that simulates on-chain multisig wallet governance entirely off-chain. It lets a group of Solana public key holders create a shared "vault," propose actions against it (transfers, data mutations, memos), and then require a configurable *threshold* of cryptographic approvals (Ed25519 signatures) before any proposal is executed. It's a trust-minimized governance engine: no single signer can act alone.

---

**Q: Why would someone build this? What problem does it solve?**

**A:** In the real world, treasuries, DAOs, and teams managing shared funds can't afford to have one person hold all the keys. A multisig enforces a rule like *"3-out-of-5 signers must agree before any money moves."* This project is a self-contained simulation of that concept — it derives deterministic Solana PDA (Program Derived Address) vault addresses, enforces threshold-based approval via real Ed25519 signature verification, and manages proposal lifecycles (pending → executed or cancelled). It's the kind of backend you'd build to prototype or test a Solana multisig program's off-chain companion service.

---

**Q: What's the tech stack?**

**A:**

| Layer | Technology | Why |
|-------|-----------|-----|
| Runtime | Node.js via `tsx` | Run TypeScript directly, zero compile step |
| Language | TypeScript (strict mode) | Type safety across the entire codebase |
| Framework | Express 4 | Lightweight, battle-tested HTTP routing |
| Crypto | `tweetnacl` + `bs58` | Ed25519 signature verification, Base58 encoding |
| Blockchain SDK | `@solana/web3.js` v1 | PublicKey validation, PDA derivation |
| Storage | In-memory `Map` objects | Stateless prototype — no database needed |

---

## Project Setup & Configuration

**Q: Walk me through `package.json`. What's notable?**

**A:** Let's break it apart:

```json
{
  "name": "multisig-vault",
  "version": "1.0.0",
  "private": true,
  "type": "commonjs",
  "scripts": {
    "start": "tsx src/server.ts"
  }
}
```

A few key choices:

- **`"type": "commonjs"`** — We're using CommonJS modules (`require`/`module.exports` under the hood), which is the default Node.js module system. The `tsconfig.json` mirrors this with `"module": "CommonJS"`.
- **`"private": true`** — This package is not meant to be published to npm. It's a safety guard.
- **`tsx` as the runner** — Instead of compiling TypeScript to JavaScript first, `tsx` interprets `.ts` files directly. This eliminates a build step entirely — you type `npm start` and the server is running.

---

**Q: And `tsconfig.json` — what's the compiler doing?**

**A:**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "outDir": "dist"
  },
  "include": ["src/**/*.ts"]
}
```

The standout here is **`"strict": true`**. This enables *all* strict type-checking flags: `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, and more. Every variable, parameter, and return type must be accounted for. In a security-sensitive project like a multisig, this is non-negotiable — a loosely-typed `any` slipping through could mean a signature check gets bypassed.

`"target": "ES2022"` means the emitted JavaScript can use modern features like top-level `await`, `Array.prototype.at()`, and `Object.hasOwn()`.

---

## The Type System — Data Modeling

**Q: Let's look at `types.ts`. How is the data modeled?**

**A:** The type system is the blueprint of the entire application. Every data structure the API produces or consumes is defined here. Let me walk through each one:

### Proposal Actions — The "What"

```typescript
export type ProposalAction = "transfer" | "set_data" | "memo";
```

A proposal can do exactly three things:

1. **`transfer`** — Move SOL to a recipient address
2. **`set_data`** — Write a key-value pair into the vault's data store
3. **`memo`** — Record a human-readable note (no side effects)

This is a **discriminated union** — the action type determines which parameter shape is valid.

---

### Proposal Parameters — The "How"

```typescript
export type TransferParams = { to: string; amount: number };
export type SetDataParams  = { key: string; value: string };
export type MemoParams     = { content: string };

export type ProposalParams = TransferParams | SetDataParams | MemoParams;
```

Each action type has its own parameter shape. `ProposalParams` is a union of all three. The validation layer (in `utils.ts`) ensures that the *correct* param shape matches the *correct* action type — you can't submit `{ content: "hello" }` for a `transfer` action.

---

### The Proposal — The Governance Unit

```typescript
export type Proposal = {
  id: number;
  vaultId: number;
  proposer: string;          // Base58 public key of who created it
  action: ProposalAction;
  params: ProposalParams;
  status: ProposalStatus;     // "pending" | "executed" | "cancelled"
  signatures: ProposalSignature[];
  createdAt: string;          // ISO 8601 timestamp
  executedAt?: string;        // Only set when status becomes "executed"
};
```

This is the heart of the system. A proposal is:
- Created by a **proposer** (who must be a vault signer)
- Born in **`"pending"`** status
- Collects **signatures** over time
- Flips to **`"executed"`** once threshold signatures are gathered, or **`"cancelled"`** if the proposer revokes it

The `executedAt` field is *optional* (`?`) — it only appears in the response once execution happens. The serializer in `utils.ts` deliberately omits it from the JSON when it's `undefined`.

---

### The Vault — The Container

```typescript
export type Vault = {
  id: number;
  label: string;              // Human-readable name
  address: string;            // Deterministic PDA address
  threshold: number;          // How many signatures needed
  bump: number;               // PDA bump seed
  signers: string[];          // Sorted array of Base58 public keys
  signerSetKey: string;       // Signers joined by ":" — used as dedup key
  createdAt: string;
  proposals: Proposal[];      // All proposals ever created for this vault
  data: Record<string, string>; // Key-value store mutated by set_data proposals
};
```

The vault is the top-level entity. Key design decisions:

- **`signers` are sorted** — This ensures deterministic PDA derivation regardless of the order signers are provided during creation.
- **`signerSetKey`** — A canonical string representation of the signer set (e.g., `"AAA...111:BBB...222:CCC...333"`). Used to enforce the rule: *one vault per unique set of signers*.
- **`data`** — A generic key-value store. When a `set_data` proposal is executed, it writes directly into this map. This makes the vault programmable beyond just transfers.

---

## Constants — The Anchor Points

**Q: What's in `constants.ts`? It's tiny — is it even worth discussing?**

**A:** Absolutely. Small files often carry outsized importance:

```typescript
import { SystemProgram } from "@solana/web3.js";

export const PORT = 3000;
export const PROGRAM_ID = SystemProgram.programId;
```

- **`PORT`** — Centralized so changes propagate everywhere. No magic `3000` scattered through the code.
- **`PROGRAM_ID`** — This is `11111111111111111111111111111111` (the Solana System Program). It's used as the **program ID seed** for PDA derivation. In a production Solana program, this would be your custom program's ID. Using the System Program here is a deliberate simulation choice — it gives us a real, valid Solana program ID to derive deterministic addresses against.

---

## The Store — In-Memory State Management

**Q: How does the application manage state? Is there a database?**

**A:** No database. Everything lives in memory:

```typescript
export const vaultsById = new Map<number, Vault>();
export const vaultIdBySignerSet = new Map<string, number>();

let nextVaultId = 1;
let nextProposalId = 1;
```

This is a **dual-index store**:

1. **`vaultsById`** — Primary index. Look up any vault by its numeric ID in O(1).
2. **`vaultIdBySignerSet`** — Secondary index. Maps a canonical signer-set string to a vault ID. This enforces the uniqueness constraint: *no two vaults can have the same set of signers*.

### ID Allocation

```typescript
export function allocateVaultId(): number {
  const id = nextVaultId;
  nextVaultId += 1;
  return id;
}

export function allocateProposalId(): number {
  const id = nextProposalId;
  nextProposalId += 1;
  return id;
}
```

Sequential, monotonically increasing IDs. Simple and predictable. Proposal IDs are **globally unique** across all vaults — not scoped per vault. This means Proposal #5 exists in exactly one vault, period.

---

**Q: What happens when the server restarts?**

**A:** Everything is lost. This is intentional — it's a prototype/contest submission, not a production database. The in-memory `Map` approach means zero setup, zero dependencies, and instant cold starts. If you wanted persistence, you'd swap these Maps for a database adapter with the exact same interface.

---

## Utilities — The Engine Room

**Q: `utils.ts` is the largest file. What's in there?**

**A:** This file contains all the business logic primitives — validation, cryptography, serialization, and execution. Think of it as the engine that the routes simply *invoke*. Let's go function by function.

---

### Error Handling

```typescript
export function errorResponse(res: Response, status: number, error: string) {
  return res.status(status).json({ error });
}
```

Every error in the API follows the `{ "error": "message" }` shape. This tiny wrapper ensures consistency — no route can accidentally return a different error format.

---

### Input Validation

**Q: How are public keys validated?**

**A:**

```typescript
export function normalizeAndValidateSigner(value: unknown): string | null {
  if (!isNonEmptyString(value)) return null;
  try {
    return new PublicKey(value).toBase58();
  } catch {
    return null;
  }
}
```

This does two things:
1. **Validates** that the string is a legitimate Solana public key (Base58-encoded, 32 bytes)
2. **Normalizes** it by round-tripping through `PublicKey` — this handles edge cases where different Base58 representations could encode the same key bytes

If *anything* is wrong, it returns `null`. No exceptions leak to the caller. This is a pattern used throughout: **validation functions return `null` on failure, never throw**.

---

**Q: How are signers deduplicated and sorted?**

**A:**

```typescript
export function getSortedUniqueSigners(signers: unknown): string[] | null {
  if (!Array.isArray(signers)) return null;

  const normalized: string[] = [];
  for (const signer of signers) {
    const pubkey = normalizeAndValidateSigner(signer);
    if (!pubkey) return null;
    normalized.push(pubkey);
  }

  const unique = new Set(normalized);
  if (unique.size !== normalized.length) return null; // duplicates found

  return [...unique].sort((a, b) => a.localeCompare(b));
}
```

This function is critical for determinism. Here's the process:

1. **Validate** every element — if *any* signer is invalid, the whole array is rejected
2. **Detect duplicates** — if the Set is smaller than the array, someone passed the same key twice → reject
3. **Sort lexicographically** — ensures that `[B, A, C]` and `[C, A, B]` produce the same sorted array → same PDA address

Why not silently dedup? Because passing duplicates is likely a client bug, and silently swallowing it could lead to a threshold misconfiguration (e.g., user thinks they have 3 signers but only 2 are unique).

---

### PDA Derivation — The Deterministic Address

**Q: This is the Solana-specific part. How does vault address derivation work?**

**A:**

```typescript
export function deriveVaultAddress(sortedSigners: string[]) {
  const signerHash = crypto
    .createHash("sha256")
    .update(sortedSigners.join(":"))
    .digest();

  const [address, bump] = PublicKey.findProgramAddressSync(
    [Buffer.from("vault"), signerHash],
    PROGRAM_ID,
  );

  return { address: address.toBase58(), bump };
}
```

This is a two-step process:

**Step 1: Hash the signer set**
The sorted signers are joined with `:` and SHA-256 hashed. This produces a fixed-size 32-byte digest regardless of how many signers there are. The hash serves as a deterministic "fingerprint" of the signer group.

**Step 2: Derive the PDA**
Solana's `findProgramAddressSync` takes:
- **Seeds**: `["vault", <signer_hash>]` — a static prefix plus the dynamic signer fingerprint
- **Program ID**: The System Program (acting as our program)

It then searches for a public key that does **not** lie on the Ed25519 curve (hence "Program Derived" — no private key exists). The `bump` is the seed byte that makes this work (it tries 255 down to 0 until it finds a valid off-curve point).

**The result:** The same set of signers will *always* produce the same vault address, regardless of the order they were originally provided.

---

### Serialization

**Q: Why are there separate serialization functions?**

**A:**

```typescript
export function serializeVault(vault: Vault) {
  return {
    id: vault.id,
    label: vault.label,
    address: vault.address,
    threshold: vault.threshold,
    bump: vault.bump,
    signers: vault.signers,
    createdAt: vault.createdAt,
  };
}
```

Notice what's **excluded**: `signerSetKey`, `proposals`, and `data`. These are internal implementation details. The API consumer doesn't need to see the signer set key (it's an internal dedup mechanism), and proposals have their own dedicated endpoints.

```typescript
export function serializeProposal(proposal: Proposal) {
  return {
    ...proposal fields...,
    ...(proposal.executedAt ? { executedAt: proposal.executedAt } : {}),
  };
}
```

The spread trick `...(condition ? { key: value } : {})` is an elegant way to *conditionally include a field*. If `executedAt` is undefined, the field doesn't appear in the JSON at all — not even as `null`. This is cleaner API design.

---

### Cryptographic Signature Verification

**Q: How are approvals verified cryptographically?**

**A:**

```typescript
export function verifyDetachedSignature(
  signer: string,
  signature: string,
  message: string,
): boolean {
  try {
    const publicKeyBytes = new PublicKey(signer).toBytes();
    const signatureBytes = bs58.decode(signature);
    const messageBytes = Buffer.from(message, "utf8");
    return nacl.sign.detached.verify(messageBytes, signatureBytes, publicKeyBytes);
  } catch {
    return false;
  }
}
```

This is real Ed25519 signature verification — the same cryptography Solana uses on-chain. Here's what happens:

1. The **signer's public key** is converted from Base58 to raw bytes
2. The **signature** is decoded from Base58 to 64 raw bytes
3. The **message** is the UTF-8 encoding of a structured string like `"approve:5"` or `"cancel:5"`
4. `nacl.sign.detached.verify` checks: *"Was this exact message signed by the private key corresponding to this public key?"*

This is a **detached** signature — the message is not embedded in the signature itself. The server reconstructs the expected message (`approve:<proposalId>` or `cancel:<proposalId>`) and verifies against it. This prevents replay attacks: a signature for proposal 5 cannot be reused for proposal 6.

---

### Proposal Validation

**Q: How does the system ensure proposals have valid parameters?**

**A:**

```typescript
export function validateProposalParams(
  action: unknown,
  params: unknown,
): { action: ProposalAction; params: ProposalParams } | null {
  if (!isPlainObject(params) || typeof action !== "string") return null;

  if (action === "transfer") {
    const to = normalizeAndValidateSigner(params.to);
    const amount = params.amount;
    if (!to || typeof amount !== "number" || !Number.isFinite(amount) || amount <= 0)
      return null;
    return { action, params: { to, amount } };
  }

  if (action === "set_data") { /* validate key + value are non-empty strings */ }
  if (action === "memo")     { /* validate content is non-empty string */ }

  return null; // unknown action
}
```

This is a **discriminated validator**. It checks the `action` string, then validates the corresponding `params` shape:

- `transfer`: `to` must be a valid Solana address, `amount` must be a finite positive number
- `set_data`: both `key` and `value` must be non-empty strings
- `memo`: `content` must be a non-empty string

If the action is unrecognized, it returns `null`. This means the API is **closed to extension by default** — you can't invent new action types without modifying this function.

---

### Proposal Execution

**Q: What actually happens when a proposal is executed?**

**A:**

```typescript
export function executeProposal(vault: Vault, proposal: Proposal) {
  if (proposal.action === "set_data") {
    const params = proposal.params as SetDataParams;
    vault.data[params.key] = params.value;
  }

  proposal.status = "executed";
  proposal.executedAt = new Date().toISOString();
}
```

Execution is the moment a proposal's side effects are applied:

- **`set_data`** → Writes the key-value pair into `vault.data`
- **`transfer`** → *No-op in this simulation* (there's no real blockchain to move SOL on)
- **`memo`** → *No-op by design* (memos are just records)

Only `set_data` has an observable effect on the vault's state. The other two action types exist to model the *proposal lifecycle* — they go through the same approval flow, they just don't mutate anything.

---

## Routes — The API Surface

**Q: Let's walk through every endpoint. What's the full API?**

**A:** The API has 7 endpoints, all mounted under `/api`:

---

### 1. `POST /api/vault/create` — Create a Vault

**Purpose:** Register a new multisig vault with a set of signers and approval threshold.

**Request Body:**
```json
{
  "signers": ["<pubkey1>", "<pubkey2>", "<pubkey3>"],
  "threshold": 2,
  "label": "Team Treasury"
}
```

**Validation Chain:**
1. `signers` → Must be an array of ≥2 valid, unique Solana public keys
2. `threshold` → Must be an integer, 1 ≤ threshold ≤ signers.length
3. `label` → Must be a non-empty string
4. **Uniqueness check** → The sorted signer set must not already have a vault (409 Conflict)

**What happens internally:**
- Signers are sorted and joined with `:` to form a `signerSetKey`
- A PDA is derived from the SHA-256 hash of the sorted signers
- A `Vault` object is created and stored in both indexes (`vaultsById` and `vaultIdBySignerSet`)

**Response (201 Created):**
```json
{
  "id": 1,
  "label": "Team Treasury",
  "address": "<deterministic PDA>",
  "threshold": 2,
  "bump": 255,
  "signers": ["<sorted pubkey1>", "<sorted pubkey2>", "<sorted pubkey3>"],
  "createdAt": "2026-04-03T12:00:00.000Z"
}
```

---

### 2. `GET /api/vault/:vaultId` — Get Vault Details

**Purpose:** Retrieve a vault's metadata, including how many proposals it has.

**Response (200 OK):**
```json
{
  "id": 1,
  "label": "Team Treasury",
  "address": "<PDA>",
  "threshold": 2,
  "bump": 255,
  "signers": ["..."],
  "createdAt": "...",
  "proposalCount": 3
}
```

Notice: `proposalCount` is computed on-the-fly from `vault.proposals.length`. The proposals themselves aren't included — you fetch those from dedicated endpoints.

---

### 3. `POST /api/vault/:vaultId/propose` — Create a Proposal

**Purpose:** A vault signer proposes an action for the group to vote on.

**Request Body:**
```json
{
  "proposer": "<signer pubkey>",
  "action": "transfer",
  "params": { "to": "<recipient>", "amount": 100 }
}
```

**Validation Chain:**
1. Vault must exist (404)
2. `proposer` must be a valid public key (400)
3. `proposer` must be a member of this vault's signer set (403)
4. `action` + `params` must pass discriminated validation (400)

**Key detail:** Creating a proposal does **not** automatically count as an approval. The proposer must separately call the approve endpoint and provide a cryptographic signature. This is a deliberate security choice — proposing and approving are distinct governance actions.

---

### 4. `POST /api/vault/:vaultId/proposals/:proposalId/approve` — Approve a Proposal

**Purpose:** A vault signer cryptographically approves a pending proposal.

**Request Body:**
```json
{
  "signer": "<pubkey>",
  "signature": "<base58 Ed25519 signature of 'approve:<proposalId>'>"
}
```

**Validation Chain (in order):**
1. Vault exists (404)
2. Proposal exists within this vault (404)
3. Proposal is not already `executed` (409) or `cancelled` (409)
4. `signer` is a valid public key (400)
5. `signer` is a member of the vault (403)
6. `signer` hasn't already signed this proposal (409)
7. `signature` is a non-empty string (400)
8. Ed25519 verification of `"approve:<proposalId>"` passes (400)

**Auto-execution:** After recording the signature, the route checks:
```typescript
if (proposal.signatures.length >= vault.threshold) {
  executeProposal(vault, proposal);
}
```
The moment enough signatures accumulate, the proposal is executed *immediately and automatically*. There is no separate "execute" step — reaching threshold *is* execution.

---

### 5. `GET /api/vault/:vaultId/proposals` — List Proposals

**Purpose:** Retrieve all proposals for a vault, optionally filtered by status.

**Query Parameters:**
- `?status=pending` — Only pending proposals
- `?status=executed` — Only executed proposals
- `?status=cancelled` — Only cancelled proposals
- *(no filter)* — All proposals

Invalid status values return a 400 error rather than being silently ignored. This is defensive design — typos like `?status=pendng` are caught.

---

### 6. `GET /api/vault/:vaultId/proposals/:proposalId` — Get Single Proposal

**Purpose:** Fetch full details of a specific proposal, including its signature list and execution timestamp.

---

### 7. `POST /api/vault/:vaultId/proposals/:proposalId/cancel` — Cancel a Proposal

**Purpose:** The original proposer revokes their proposal before it's executed.

**Request Body:**
```json
{
  "signer": "<proposer pubkey>",
  "signature": "<base58 Ed25519 signature of 'cancel:<proposalId>'>"
}
```

**Critical restriction:** Only the **original proposer** can cancel. Not any signer — just the person who created it. This prevents griefing where a minority signer could kill proposals they disagree with.

The cancel message is `"cancel:<proposalId>"` — distinct from the approval message `"approve:<proposalId>"`. This means a signature for approving cannot be replayed to cancel, and vice versa.

---

### 8. `GET /api/vault/:vaultId/data` — Read Vault Data Store

**Purpose:** Retrieve the key-value data that has been written by executed `set_data` proposals.

**Response:**
```json
{
  "data": {
    "config.fee": "0.01",
    "config.name": "Main Treasury"
  }
}
```

This endpoint is read-only. The *only* way to write to `data` is through the proposal governance flow. This is the multisig's value proposition: even simple data changes require group consensus.

---

## The Server — Tying It All Together

**Q: What does `server.ts` actually do?**

**A:**

```typescript
const app = express();

app.use(express.json());        // Parse JSON request bodies
app.use("/api", vaultRouter);   // Mount all routes under /api

app.use((_req, res) => {        // Catch-all 404
  return res.status(404).json({ error: "Not found" });
});

app.listen(PORT, () => {
  console.log(`Multi-sig vault server listening on port ${PORT}`);
});
```

This is deliberately minimal — 18 lines. The server file's job is:

1. **Wire middleware** — JSON body parsing
2. **Mount routes** — Everything under `/api`
3. **Handle 404s** — Any unmatched route gets a JSON error (not an HTML "Cannot GET" page)
4. **Start listening** — On port 3000

The separation is important: `server.ts` knows *nothing* about vaults, proposals, or cryptography. It's pure infrastructure. You could swap Express for Fastify by changing only this file.

---

## Security Model

**Q: What prevents abuse? Walk me through the security layers.**

**A:** The system has multiple overlapping security controls:

### 1. Signer Membership Enforcement
Every mutation (propose, approve, cancel) checks that the acting public key is in `vault.signers`. You can't propose to a vault you don't belong to, and you can't approve someone else's vault's proposals.

### 2. Cryptographic Signature Verification
Approvals and cancellations require an **Ed25519 signature** over a structured message. The server doesn't trust the `signer` field alone — it verifies that the signer actually possesses the corresponding private key. This is the same signature scheme Solana uses for transaction signing.

### 3. Replay Protection
Signatures are bound to a specific proposal ID:
- Approval: `"approve:5"`
- Cancellation: `"cancel:5"`

A signature for proposal 5 cannot approve proposal 6. An approval signature cannot cancel. The message structure makes cross-action and cross-proposal replay impossible.

### 4. Double-Signature Prevention
```typescript
if (proposal.signatures.some((entry) => entry.signer === signer)) {
  return errorResponse(res, 409, "Already signed");
}
```
A signer can only approve once per proposal. This prevents a single signer from meeting the threshold alone by submitting multiple approvals.

### 5. State Machine Guards
Proposals follow a strict state machine:
```
pending → executed  (via reaching threshold)
pending → cancelled (via proposer cancellation)
```
Once a proposal is `executed` or `cancelled`, no further actions are possible. The API returns 409 Conflict for any attempt to approve or cancel a finalized proposal.

### 6. Proposer-Only Cancellation
Only the original proposer can cancel. This prevents minority signers from blocking governance by cancelling proposals they disagree with.

### 7. Signer Set Uniqueness
The `vaultIdBySignerSet` index prevents creating duplicate vaults for the same group. This avoids confusion and potential governance fragmentation.

### 8. Input Normalization
All public keys are round-tripped through `PublicKey.toBase58()`, ensuring canonical representation. This prevents subtle bugs where different Base58 encodings of the same key could bypass membership checks.

---

## How a Full Workflow Plays Out

**Q: Can you walk through a complete real-world scenario?**

**A:** Let's say Alice, Bob, and Charlie want to manage a shared treasury with a 2-of-3 threshold.

### Step 1: Create the Vault
```
POST /api/vault/create
{
  "signers": [alice_pubkey, bob_pubkey, charlie_pubkey],
  "threshold": 2,
  "label": "Team Treasury"
}
```
→ Signers are sorted, hashed, PDA derived → Vault #1 created

### Step 2: Alice Proposes a Transfer
```
POST /api/vault/1/propose
{
  "proposer": alice_pubkey,
  "action": "transfer",
  "params": { "to": vendor_pubkey, "amount": 500 }
}
```
→ Proposal #1 created in `"pending"` status, 0 signatures

### Step 3: Alice Approves Her Own Proposal
```
POST /api/vault/1/proposals/1/approve
{
  "signer": alice_pubkey,
  "signature": "<alice signs 'approve:1' with her private key>"
}
```
→ 1 of 2 required signatures recorded. Still pending.

### Step 4: Bob Approves
```
POST /api/vault/1/proposals/1/approve
{
  "signer": bob_pubkey,
  "signature": "<bob signs 'approve:1' with his private key>"
}
```
→ 2 of 2 required signatures reached → **Proposal auto-executes**

### Step 5: Check the Result
```
GET /api/vault/1/proposals/1
```
→ Status: `"executed"`, `executedAt` timestamp present, both signatures listed

### Alternative: Alice Cancels Before Threshold
If Alice changes her mind before Bob signs:
```
POST /api/vault/1/proposals/1/cancel
{
  "signer": alice_pubkey,
  "signature": "<alice signs 'cancel:1'>"
}
```
→ Status: `"cancelled"`. Bob can no longer approve.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────┐
│                    server.ts                        │
│            Express app + JSON middleware            │
│                 + 404 catch-all                     │
└──────────────────────┬──────────────────────────────┘
                       │  app.use("/api", vaultRouter)
                       ▼
┌─────────────────────────────────────────────────────┐
│                    routes.ts                        │
│         7 endpoints: CRUD + governance flow         │
│     POST create │ GET vault │ POST propose          │
│     POST approve │ GET proposals │ GET proposal     │
│     POST cancel │ GET data                          │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
           ▼                          ▼
┌──────────────────────┐  ┌───────────────────────────┐
│      store.ts        │  │        utils.ts           │
│  vaultsById (Map)    │  │  Validation & Normalization│
│  vaultIdBySignerSet  │  │  PDA Derivation            │
│  ID allocators       │  │  Ed25519 Verification      │
└──────────────────────┘  │  Serialization             │
                          │  Proposal Execution        │
                          └──────────┬────────────────┘
                                     │
                          ┌──────────┴────────────┐
                          │                       │
                          ▼                       ▼
                   ┌────────────┐          ┌───────────┐
                   │ types.ts   │          │constants.ts│
                   │ Data models│          │ PORT       │
                   │ Vault      │          │ PROGRAM_ID │
                   │ Proposal   │          └───────────┘
                   │ Actions    │
                   └────────────┘
```

---

## Final Thoughts

**Q: If you had to summarize the design philosophy in three principles, what would they be?**

**A:**

1. **Fail loud, fail early.** Every input is validated exhaustively. Invalid public keys, duplicate signers, wrong action params — all rejected at the gate with clear error messages. Nothing ambiguous makes it through.

2. **Crypto is not optional.** Approvals aren't just "I say I'm Alice" — they require proving you hold Alice's private key. The same Ed25519 scheme used by Solana itself is used here. This is real security, not theater.

3. **Separation of concerns is separation of risk.** Types know nothing about HTTP. Utils know nothing about Express routing. Routes know nothing about how PDAs are derived. Each layer has a single responsibility, and bugs in one layer are contained.

---

> *This document covers every file, every function, and every design decision in the Multisig Vault project. For specific code references, see the source files in `src/`.*
