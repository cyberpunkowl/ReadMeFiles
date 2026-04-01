# 🧠 Solana PDA Name Registry — In-Depth Project Breakdown

> **Format:** Interview-style Q\&A — covering architecture, every module, core Solana concepts, and design decisions.

---

## 📋 Table of Contents

| # | Section |
|---|---------|
| 1 | [Project Overview](#-1-project-overview) |
| 2 | [What is a PDA?](#-2-what-is-a-pda) |
| 3 | [Tech Stack & Dependencies](#-3-tech-stack--dependencies) |
| 4 | [Project Architecture](#-4-project-architecture) |
| 5 | [The Data Model — `types.ts`](#-5-the-data-model--typests) |
| 6 | [In-Memory Database — `db.ts`](#-6-in-memory-database--dbts) |
| 7 | [Utility Functions — `utils.ts`](#-7-utility-functions--utilsts) |
| 8 | [Name Registration — `routes/registry.ts`](#-8-name-registration--routesregistryts) |
| 9 | [Sub-Name Registration — `routes/sub.ts`](#-9-sub-name-registration--routessubts) |
| 10 | [Ownership Transfer — `routes/transfer.ts`](#-10-ownership-transfer--routestransferts) |
| 11 | [PDA Verification — `routes/verify.ts`](#-11-pda-verification--routesverifyts) |
| 12 | [Listing Names — `routes/list.ts`](#-12-listing-names--routeslistts) |
| 13 | [App Entry Point — `index.ts`](#-13-app-entry-point--indexts) |
| 14 | [Testing — `test-api.ts`](#-14-testing--test-apits) |
| 15 | [Security & Cryptography Deep Dive](#-15-security--cryptography-deep-dive) |
| 16 | [Error Handling Strategy](#-16-error-handling-strategy) |
| 17 | [Potential Improvements](#-17-potential-improvements) |

---

## 🎯 1. Project Overview

**Q: What exactly is this project?**

> This is a **REST API server** that implements a **PDA-based Name Registry** on top of Solana primitives. Think of it like a **domain name system (DNS)**, but built using Solana's Program Derived Addresses (PDAs). Users can:
>
> - **Register** human-readable names (like `"alice"`, `"myapp"`) and get a deterministic PDA address derived from that name.
> - **Create sub-names** under a parent (like `"payments"` under `"myapp"`) — forming a hierarchy.
> - **Transfer ownership** of names securely using Ed25519 cryptographic signatures.
> - **Verify** whether a given address is a valid PDA derived from specific seeds.
> - **List** all registered names and sub-names.

**Q: Why would you need something like this?**

> On Solana, PDAs are addresses derived from a program ID and a set of seeds — they're not random. This registry gives those PDAs **human-readable names**, making it easier to discover and reference on-chain accounts. It's analogous to how ENS (Ethereum Name Service) works, but modeled around Solana's PDA derivation.

**Q: Does this interact with Solana mainnet/devnet?**

> **No.** This server runs entirely **off-chain**. It uses `@solana/web3.js` purely for its **cryptographic utilities** — PDA derivation (`findProgramAddressSync`) and public key validation. No RPC calls are made to any Solana cluster. The data is stored in-memory on the server.

---

## 🔑 2. What is a PDA?

**Q: Can you explain what a PDA (Program Derived Address) is?**

> A PDA is a **deterministic address** on Solana that is derived from:
>
> 1. A set of **seeds** (arbitrary byte arrays — strings, public keys, etc.)
> 2. A **program ID** (the public key of the Solana program)
>
> The derivation uses `SHA-256` hashing internally. The key property: **PDAs do not have a corresponding private key**, so no one can sign transactions as a PDA. Only the program that owns it can "sign" for it through CPI (Cross-Program Invocation).

**Q: What is the `bump` seed?**

> Not every combination of seeds produces a valid PDA (it might land on the Ed25519 curve, which means it *would* have a private key). Solana solves this by appending a **bump seed** — a single byte starting at 255 and decrementing until the result falls **off** the curve.
>
> `findProgramAddressSync` does this automatically and returns both the PDA address and the bump value.
>
> ```typescript
> const [pda, bump] = PublicKey.findProgramAddressSync(
>     [Buffer.from("name"), Buffer.from("alice")],
>     new PublicKey(programId)
> );
> // pda  = the derived address (a base58 string)
> // bump = the bump seed used (e.g. 253)
> ```

**Q: Why store the bump?**

> The bump is stored so that anyone can **re-derive** the PDA later without brute-forcing. On-chain programs require the bump to reconstruct the PDA during transaction validation. Storing it here makes re-verification trivial.

---

## 🛠️ 3. Tech Stack & Dependencies

**Q: What technologies does this project use?**

> | Package | Purpose |
> |---------|---------|
> | **Express** (`express`) | HTTP server and routing framework |
> | **@solana/web3.js** | PDA derivation (`findProgramAddressSync`) and `PublicKey` validation |
> | **bs58** | Base58 encoding/decoding (Solana addresses are Base58-encoded) |
> | **tweetnacl** | Ed25519 signature verification for ownership transfers |
> | **tsx** | TypeScript execution without a separate compile step |
> | **typescript** | Type safety and interfaces |

**Q: Why `tweetnacl` instead of `@solana/web3.js` for signature verification?**

> `tweetnacl` is a lightweight, audited NaCl (Networking and Cryptography Library) implementation. Solana wallets use **Ed25519** for signing — `tweetnacl`'s `nacl.sign.detached.verify()` maps directly to this. Using it avoids pulling in heavier Solana transaction-signing infrastructure when all we need is raw signature verification.

**Q: Why `bs58`?**

> Solana addresses and signatures are encoded in **Base58Check** format (like Bitcoin). `bs58` handles the conversion between raw bytes and the human-readable Base58 string format. For example:
> - Public key bytes (32 bytes) → `"11111111111111111111111111111111"` (Base58)
> - Signature bytes (64 bytes) → Base58 string

---

## 🏗️ 4. Project Architecture

**Q: How is the code organized?**

> The project follows a **modular architecture** with clear separation of concerns:
>
> ```
> src/
> ├── index.ts              ← App entry point: creates Express app, mounts routers
> ├── types.ts              ← Shared TypeScript interface (Registration)
> ├── db.ts                 ← In-memory database (single shared array)
> ├── utils.ts              ← Reusable validation helpers
> ├── test-api.ts           ← End-to-end test script
> └── routes/
>     ├── registry.ts       ← POST /register, GET /resolve
>     ├── sub.ts            ← POST /sub/register, GET /sub/resolve
>     ├── transfer.ts       ← POST /transfer
>     ├── verify.ts         ← POST /verify
>     └── list.ts           ← GET /list (top-level & sub-names)
> ```

**Q: Why split into separate route files?**

> Several reasons:
> 1. **Single Responsibility** — Each file handles one domain concern (registration, transfer, verification, etc.)
> 2. **Scalability** — Adding new routes (e.g., `/api/registry/revoke`) means adding a new file, not bloating an existing one.
> 3. **Testability** — Individual routers can be unit-tested in isolation by mounting them on a test Express app.
> 4. **Readability** — A developer can jump directly to `routes/transfer.ts` to understand ownership transfer without wading through unrelated code.

**Q: How do the routes share data?**

> Through two shared modules:
> - **`db.ts`** exports a single `Registration[]` array. Since JavaScript/Node.js modules are **singletons**, every route file that imports `db` gets a reference to the **same** array instance.
> - **`types.ts`** exports the `Registration` interface so every file uses the same shape.

---

## 📦 5. The Data Model — `types.ts`

**Q: What does the `Registration` interface look like and what does each field mean?**

> ```typescript
> export interface Registration {
>     id: string;            // UUID v4 — unique identifier for this registration
>     name: string;          // The human-readable name (e.g. "alice", "myapp")
>     parentName?: string | null;  // Parent's name (only for sub-names)
>     parentPda?: string | null;   // Parent's PDA address (only for sub-names)
>     programId: string;     // The Solana program ID used for PDA derivation
>     pda: string;           // The derived PDA address (Base58)
>     owner: string;         // Current owner's public key (Base58)
>     bump: number;          // The bump seed used in PDA derivation
>     createdAt: string;     // ISO 8601 timestamp
> }
> ```

**Q: Why are `parentName` and `parentPda` nullable?**

> Because the same interface is used for **both** top-level names and sub-names:
> - **Top-level names** (e.g. `"myapp"`) have `parentName: null` and `parentPda: null`.
> - **Sub-names** (e.g. `"payments"` under `"myapp"`) have `parentName: "myapp"` and `parentPda: "<parent's PDA address>"`.
>
> This avoids having two separate interfaces and simplifies database queries — you can distinguish top-level from sub-names by checking `!r.parentPda`.

**Q: Why store `programId` per registration?**

> The same name (e.g. `"alice"`) can be registered under **different programs**. Each program has its own namespace. The `programId` effectively acts as a **namespace separator**, so `"alice"` under Program A and `"alice"` under Program B are completely different registrations with different PDAs.

---

## 💾 6. In-Memory Database — `db.ts`

**Q: How does the database work?**

> ```typescript
> import { Registration } from "./types";
> export const db: Registration[] = [];
> ```
>
> It's a plain JavaScript array. That's it.

**Q: Why an in-memory array instead of a real database?**

> This is a **demonstration/prototype** API. An in-memory store:
> - Has **zero setup** — no database server, no connection strings, no migrations.
> - Is **fast** — array lookups are negligible for demo workloads.
> - Keeps the focus on **business logic** (PDA derivation, signature verification) rather than database boilerplate.
>
> The trade-off: all data is lost when the server restarts.

**Q: Why export it as a `const`? Can it still be modified?**

> Yes. `const` in JavaScript means the **binding** can't be reassigned (`db = somethingElse` would error), but the array's **contents** can still be mutated (`db.push(...)`, `db[i].owner = ...`). This is intentional — the routes need to add items and modify ownership.

---

## 🔧 7. Utility Functions — `utils.ts`

**Q: What does `isValidBase58` do?**

> ```typescript
> export function isValidBase58(str: string): boolean {
>     if (!str) return false;
>     try {
>         const decoded = bs58.decode(str);
>         return decoded.length === 32;
>     } catch {
>         return false;
>     }
> }
> ```
>
> It validates that a string is a **valid Solana public key** by checking two things:
> 1. The string is valid **Base58** (the `decode` doesn't throw).
> 2. The decoded bytes are exactly **32 bytes** long (the size of an Ed25519 public key).

**Q: Why not just use `new PublicKey(str)` and catch errors?**

> That would work too, but `PublicKey` construction does more than just validation — it creates an object with methods. Using `bs58.decode` is a **lighter-weight check** that doesn't instantiate unnecessary objects. It's also reusable outside of Solana-specific contexts.

**Q: Why is this function shared in `utils.ts`?**

> It's used in **four different route files** (registry, sub, transfer, verify). Extracting it avoids code duplication and ensures consistent validation logic across all endpoints.

---

## 📝 8. Name Registration — `routes/registry.ts`

**Q: Walk me through the registration flow.**

> **Endpoint:** `POST /api/registry/register`
>
> **Step-by-step:**
>
> 1. **Extract** `name`, `programId`, and `owner` from the request body.
> 2. **Validate** that all fields are present and that `programId` and `owner` are valid Base58 public keys.
> 3. **Check for duplicates** — search the database for an existing top-level registration with the same `name` and `programId`.
> 4. **Derive the PDA** using `findProgramAddressSync` with seeds `["name", <the-name>]` and the given `programId`.
> 5. **Create** a `Registration` object with a UUID, the derived PDA, bump, and current timestamp.
> 6. **Store** it in the database and return it with status `201 Created`.

**Q: Why are the seeds `["name", <the-name>]`?**

> The string `"name"` acts as a **seed prefix/discriminator**. This is a Solana convention to namespace different types of PDAs within the same program. For example:
> - Name PDAs: `["name", "alice"]`
> - Sub-name PDAs: `["sub", <parent_pda_bytes>, "child"]`
>
> Without the prefix, a name and a sub-name could accidentally collide if they shared the same raw seed bytes.

**Q: What about the resolve endpoint?**

> **Endpoint:** `GET /api/registry/resolve/:programId/:name`
>
> This is a simple **lookup** — find the registration where `name` and `programId` match and `parentPda` is null (i.e., it's a top-level name). Returns `404` if not found.

---

## 🌳 9. Sub-Name Registration — `routes/sub.ts`

**Q: How do sub-names differ from top-level names?**

> Sub-names create a **hierarchy** — like directories under a root:
>
> ```
> myapp                    ← top-level (parent)
> ├── payments             ← sub-name
> ├── users                ← sub-name
> └── settings             ← sub-name
> ```
>
> The key differences in PDA derivation:
>
> | | Top-level | Sub-name |
> |---|-----------|----------|
> | **Seeds** | `["name", <name>]` | `["sub", <parent_pda_bytes>, <sub_name>]` |
> | **Parent reference** | `null` | Parent's PDA address |
> | **Uniqueness** | Unique per `(name, programId)` | Unique per `(subName, parentPda, programId)` |

**Q: Why use the parent's PDA bytes as a seed?**

> This **cryptographically binds** the sub-name to its parent. The sub-name's PDA is derived from:
> ```typescript
> [Buffer.from("sub"), new PublicKey(parent.pda).toBuffer(), Buffer.from(subName)]
> ```
>
> This means:
> - `"payments"` under Parent A → different PDA than `"payments"` under Parent B.
> - You can't fake a parent relationship — the PDA derivation would produce a different address.

**Q: What validations happen before creating a sub-name?**

> 1. All required fields (`parentName`, `subName`, `programId`, `owner`) must be present.
> 2. `programId` and `owner` must be valid Base58 public keys.
> 3. The **parent must exist** — there must be a top-level registration with `parentName` under the given `programId`.
> 4. The **sub-name must not already exist** under the same parent.

---

## 🔄 10. Ownership Transfer — `routes/transfer.ts`

**Q: How does ownership transfer work? This seems like the most security-critical part.**

> **Endpoint:** `POST /api/registry/transfer`
>
> This is where **cryptography meets authorization**. The flow:
>
> 1. **Extract** `programId`, `name`, `newOwner`, `signature`, and `message` from the request body.
>
> 2. **Validate the message format** — it must exactly match:
>    ```
>    transfer:<name>:to:<newOwner>
>    ```
>    For example: `transfer:myapp:to:7Hk3...xQ9z`
>
> 3. **Look up** the registration in the database.
>
> 4. **Verify the Ed25519 signature:**
>    ```typescript
>    nacl.sign.detached.verify(
>        Buffer.from(message, "utf-8"),   // the raw message bytes
>        bs58.decode(signature),           // the signature (64 bytes)
>        bs58.decode(reg.owner)            // the current owner's public key (32 bytes)
>    );
>    ```
>
> 5. If valid, **update** `reg.owner` to `newOwner` and return the updated registration.

**Q: Why require a specific message format instead of letting users sign anything?**

> This prevents **replay attacks** and **misuse of signatures**:
> - The message encodes the **intent**: *what* is being transferred and *to whom*.
> - A signature for `transfer:myapp:to:Alice` cannot be reused to transfer to Bob.
> - A signature for transferring `"myapp"` cannot be reused to transfer `"other"`.
>
> Without this, a malicious actor could reuse a signature from a different context.

**Q: Walk me through the cryptography.**

> ```
> ┌──────────────────────────────────────────────────┐
> │                   SIGNING (Client-side)          │
> │                                                  │
> │  message = "transfer:myapp:to:<newOwner>"        │
> │  signature = Ed25519_Sign(message, secretKey)    │
> │  Send: { message, signature (base58), ... }      │
> └──────────────────────┬─────────────────────────────┘
>                        │
>                        ▼
> ┌──────────────────────────────────────────────────┐
> │                 VERIFICATION (Server-side)       │
> │                                                  │
> │  Lookup registration → get current owner pubkey  │
> │  Ed25519_Verify(message, signature, ownerPubkey) │
> │  If true → owner actually authorized this        │
> └──────────────────────────────────────────────────┘
> ```
>
> - **Ed25519** is an elliptic curve signature scheme.
> - Only the holder of the **private key** can produce a valid signature.
> - The server only stores the **public key** and can verify without ever knowing the private key.

**Q: What happens if the signature is invalid?**

> The endpoint returns `403 Forbidden` with `{ error: "Invalid signature" }`. This covers:
> - Wrong private key used to sign.
> - Tampered message.
> - Corrupted signature bytes.
> - The `try/catch` also handles malformed input (e.g., signature that isn't valid Base58).

---

## ✅ 11. PDA Verification — `routes/verify.ts`

**Q: What does PDA verification do?**

> **Endpoint:** `POST /api/registry/verify`
>
> It answers the question: **"Is this address actually a valid PDA derived from these specific seeds and program ID?"**
>
> ```typescript
> const seedBuffers = seeds.map(s => Buffer.from(s, "utf-8"));
> const [expectedPda, bump] = PublicKey.findProgramAddressSync(
>     seedBuffers,
>     new PublicKey(programId)
> );
>
> return {
>     valid: expectedPda.toBase58() === address,  // does it match?
>     expectedPda: expectedPda.toBase58(),         // what we computed
>     bump                                         // the bump seed
> };
> ```

**Q: Why is this useful?**

> In Solana development, you often need to verify that an address was correctly derived. This endpoint lets you:
> - **Debug** PDA derivation issues (wrong seeds? wrong program?).
> - **Audit** whether an address on-chain matches expected derivation parameters.
> - **Validate** before submitting transactions that reference a PDA.

**Q: Does this require the name to be registered first?**

> **No.** This endpoint is **stateless** — it doesn't look at the database at all. It's a pure cryptographic computation. You can verify any PDA from any seeds and program ID, regardless of whether it was registered through this API.

---

## 📋 12. Listing Names — `routes/list.ts`

**Q: How does listing work?**

> Two endpoints:
>
> ### List Top-Level Names
> **`GET /api/registry/list/:programId`**
>
> Returns all top-level registrations (where `parentPda` is null) for a given program. Optionally filters by `owner` via query parameter:
> ```
> GET /api/registry/list/<programId>?owner=<ownerPubkey>
> ```
>
> ### List Sub-Names
> **`GET /api/registry/list/:programId/:name/subs`**
>
> Returns all sub-names under a specific parent name. First verifies the parent exists, then filters registrations where `parentPda` matches the parent's PDA.

**Q: Why does listing subs require looking up the parent first?**

> Two reasons:
> 1. **Validation** — if the parent doesn't exist, return `404` instead of an empty array (an empty array would be ambiguous — does the parent have no subs, or does it not exist?).
> 2. **Correct filtering** — sub-names are linked by `parentPda` (the PDA address), not by `parentName`. The parent lookup converts the name to its PDA for accurate filtering.

---

## 🚀 13. App Entry Point — `index.ts`

**Q: What does the entry point look like after the split?**

> ```typescript
> import express from "express";
> import registryRouter from "./routes/registry";
> import subRouter from "./routes/sub";
> import transferRouter from "./routes/transfer";
> import verifyRouter from "./routes/verify";
> import listRouter from "./routes/list";
>
> const app = express();
> app.use(express.json());
>
> app.use("/api/registry", registryRouter);
> app.use("/api/registry/sub", subRouter);
> app.use("/api/registry", transferRouter);
> app.use("/api/registry", verifyRouter);
> app.use("/api/registry", listRouter);
>
> app.listen(3000, () => {
>   console.log("Server running on port 3000");
> });
> ```

**Q: How does Express router mounting work here?**

> Each route file exports an Express `Router`. The `app.use(prefix, router)` call **mounts** that router at the given prefix. The route paths inside the router are **relative** to the mount point:
>
> | Mount Point | Router Path | Final URL |
> |-------------|-------------|-----------|
> | `/api/registry` | `/register` | `/api/registry/register` |
> | `/api/registry` | `/resolve/:programId/:name` | `/api/registry/resolve/:programId/:name` |
> | `/api/registry/sub` | `/register` | `/api/registry/sub/register` |
> | `/api/registry` | `/transfer` | `/api/registry/transfer` |
> | `/api/registry` | `/verify` | `/api/registry/verify` |
> | `/api/registry` | `/list/:programId` | `/api/registry/list/:programId` |

**Q: Why is `express.json()` important?**

> It's middleware that **parses JSON request bodies**. Without it, `req.body` would be `undefined` for all POST requests. It reads the `Content-Type: application/json` header, parses the body, and populates `req.body` with the resulting object.

---

## 🧪 14. Testing — `test-api.ts`

**Q: How does the test script work?**

> It's an end-to-end integration test that exercises **every endpoint** in sequence:
>
> ```
> 1. Generate random keypairs (programId, owner, newOwner)
> 2. POST /register        → register "myname"
> 3. GET  /resolve          → resolve "myname"
> 4. POST /sub/register     → register "sub1" under "myname"
> 5. GET  /sub/resolve      → resolve "sub1"
> 6. POST /transfer         → transfer "myname" to newOwner (with real Ed25519 signature)
> 7. POST /verify           → verify the PDA matches expected derivation
> 8. GET  /list             → list all top-level names
> 9. GET  /list/.../subs    → list sub-names under "myname"
> ```

**Q: How does the test create a valid signature?**

> It uses `Keypair.generate()` from `@solana/web3.js` to create real Ed25519 keypairs, then signs with `tweetnacl`:
>
> ```typescript
> const owner = Keypair.generate();
> const transferMessage = `transfer:myname:to:${newOwner.publicKey.toBase58()}`;
> const signature = nacl.sign.detached(
>     Buffer.from(transferMessage, "utf-8"),
>     owner.secretKey    // 64-byte secret key
> );
> const signatureBase58 = bs58.encode(signature);
> ```
>
> This produces a **cryptographically valid** Ed25519 signature that the server can verify against the owner's public key.

---

## 🔐 15. Security & Cryptography Deep Dive

**Q: What cryptographic primitives does this project use?**

> | Primitive | Where | Purpose |
> |-----------|-------|---------|
> | **SHA-256** | PDA derivation (inside `findProgramAddressSync`) | Deterministically derive addresses from seeds |
> | **Ed25519** | Ownership transfer | Verify the current owner authorized the transfer |
> | **Base58** | Everywhere | Encode/decode Solana addresses and signatures |

**Q: Is the signature verification secure?**

> Yes, within the scope of this API:
> - **Authentication**: Only the private key holder can produce a valid signature → only the owner can transfer.
> - **Intent binding**: The message format `transfer:<name>:to:<newOwner>` prevents cross-purpose signature reuse.
> - **Non-repudiation**: The signature proves the owner authorized the specific transfer.
>
> **Limitations** (since this is an off-chain demo):
> - No nonce/timestamp → a signature could theoretically be replayed if ownership is transferred back.
> - No rate limiting or authentication on other endpoints.

**Q: What is Base58 and why does Solana use it?**

> Base58 is a binary-to-text encoding that uses 58 alphanumeric characters, **excluding** visually ambiguous characters (`0`, `O`, `I`, `l`). This reduces human-error when copying addresses. Solana (and Bitcoin) chose it for its balance of compactness and readability.
>
> ```
> Alphabet: 123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz
> Missing:  0, O, I, l  (easily confused characters)
> ```

---

## ⚠️ 16. Error Handling Strategy

**Q: How does the API handle errors?**

> Every endpoint follows a **consistent pattern**:
>
> | HTTP Status | Meaning | Example |
> |-------------|---------|---------|
> | `400` | Bad request — missing fields, invalid format | `{ error: "Missing or invalid fields" }` |
> | `403` | Forbidden — signature verification failed | `{ error: "Invalid signature" }` |
> | `404` | Not found — name doesn't exist | `{ error: "Name not found" }` |
> | `409` | Conflict — duplicate registration | `{ error: "Name already registered for this programId" }` |
> | `201` | Created — successful registration | Full registration object |
> | `200` | OK — successful query/transfer/verify | Response data |

**Q: Why use `try/catch` around PDA derivation?**

> `PublicKey.findProgramAddressSync` can throw if:
> - The `programId` is not a valid public key.
> - No valid PDA exists for the given seeds (extremely rare but theoretically possible).
>
> The catch block returns the error message as a `400` response, preventing the server from crashing on invalid input.

---

## 🚀 17. Potential Improvements

**Q: If this were going to production, what would you change?**

> | Area | Improvement |
> |------|-------------|
> | **Persistence** | Replace the in-memory array with PostgreSQL, SQLite, or Redis |
> | **Replay Protection** | Add nonces or timestamps to transfer messages |
> | **Rate Limiting** | Add `express-rate-limit` to prevent abuse |
> | **Authentication** | Require signed requests for all write operations |
> | **Pagination** | Add `limit`/`offset` to list endpoints |
> | **Input Sanitization** | Enforce name length limits, character restrictions |
> | **Logging** | Structured logging with `winston` or `pino` |
> | **Testing** | Unit tests with `jest` + supertest, CI pipeline |
> | **On-chain Sync** | Verify registrations against actual on-chain PDA accounts |
> | **HTTPS** | TLS termination for production deployment |

---

<div align="center">

*Built with Node.js, Express, TypeScript, and Solana Web3.js*

</div>
