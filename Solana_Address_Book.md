# 🔍 Solana Address Book API — In-Depth Project Deep Dive

> **Format:** Interview-style Q&A  
> **Purpose:** A thorough, technical walkthrough of every major component in the codebase  
> **Project:** `solana-address-book` — A RESTful API for managing a Solana address book with cryptographic utilities  

---

## 📖 Table of Contents

1. [Project Overview & Architecture](#1-project-overview--architecture)  
2. [Configuration & Tooling](#2-configuration--tooling)  
3. [Type System](#3-type-system)  
4. [Error Handling Strategy](#4-error-handling-strategy)  
5. [Validation Layer](#5-validation-layer)  
6. [Solana Service — Cryptographic Core](#6-solana-service--cryptographic-core)  
7. [Contact Service — Business Logic](#7-contact-service--business-logic)  
8. [Controllers — Request Handling](#8-controllers--request-handling)  
9. [Routing Layer](#9-routing-layer)  
10. [Application Entry Point](#10-application-entry-point)  
11. [Data Flow — End to End](#11-data-flow--end-to-end)  
12. [Security Considerations](#12-security-considerations)  

---

## 1. Project Overview & Architecture

### Q: What is this project, and what problem does it solve?

**A:** This is a **Solana Address Book API** — a REST API server built with Node.js, TypeScript, and Express. It lets users manage a collection of Solana blockchain addresses as "contacts," while also exposing powerful Solana-native cryptographic utilities like:

- **Associated Token Account (ATA) derivation** — find the token account for any wallet/mint pair
- **Program Derived Address (PDA) derivation** — compute deterministic on-chain addresses
- **Ed25519 signature verification** — prove ownership of a wallet by signing a message

Think of it as a backend service that a Solana wallet or dApp could use to maintain an address book with built-in blockchain-aware features.

---

### Q: How is the codebase structured? Walk me through the folder layout.

**A:** The project follows a clean **layered architecture** with separation of concerns:

```
src/
├── index.ts                    # Entry point — Express app setup
├── types/
│   └── contact.ts              # TypeScript interfaces (data contracts)
├── utils/
│   ├── errors.ts               # Custom error classes
│   └── validation.ts           # Input validation functions
├── services/
│   ├── contactService.ts       # Business logic for CRUD operations
│   └── solanaService.ts        # Solana-specific crypto operations
├── controllers/
│   ├── contactsController.ts   # HTTP handlers for contact endpoints
│   └── cryptoController.ts     # HTTP handlers for crypto endpoints
└── routes/
    ├── contacts.ts             # Contact route definitions
    └── crypto.ts               # Crypto route definitions
```

The flow goes: **Routes → Controllers → Services → Utils/Types**. Each layer has a single responsibility:

| Layer         | Responsibility                                      |
|---------------|-----------------------------------------------------|
| **Routes**    | Map HTTP verbs + paths to controller handlers        |
| **Controllers** | Parse requests, call services, format responses    |
| **Services**  | Core business logic, data manipulation, crypto ops   |
| **Utils**     | Cross-cutting concerns: validation, error classes    |
| **Types**     | TypeScript interfaces — the data contracts           |

This means a controller never talks to the database directly, and a route never contains business logic. Each piece is independently testable and replaceable.

---

## 2. Configuration & Tooling

### Q: What's in `package.json` and why were these specific dependencies chosen?

**A:** Let's break down every dependency:

```json
{
  "name": "solana-address-book",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "tsx src/index.ts",
    "dev": "tsx watch src/index.ts"
  }
}
```

**`"type": "module"`** — This is critical. It tells Node.js to treat `.js` files as ES Modules (not CommonJS). That's why all internal imports use the `.js` extension — TypeScript compiles `.ts` to `.js`, and Node.js ESM requires explicit extensions.

**Runtime Dependencies:**

| Package           | Version   | Purpose                                                                 |
|-------------------|-----------|-------------------------------------------------------------------------|
| `@solana/web3.js` | `^1.98.4` | The official Solana SDK — `PublicKey` class, PDA derivation, curve checks |
| `bs58`            | `^6.0.0`  | Base58 encoding/decoding — Solana addresses are base58-encoded          |
| `express`         | `^5.2.1`  | HTTP framework — note this is **Express v5**, not v4                    |
| `tweetnacl`       | `^1.0.3`  | Ed25519 signature verification — the same crypto Solana wallets use     |
| `tsx`             | `^4.19.4` | TypeScript execution engine — runs `.ts` directly without a build step  |
| `typescript`      | `^5.8.3`  | TypeScript compiler (used for type checking)                            |

**Dev Dependencies:**

| Package         | Purpose                                    |
|-----------------|--------------------------------------------|
| `@types/express`| TypeScript type definitions for Express    |
| `@types/node`   | TypeScript type definitions for Node.js    |

---

### Q: What about `tsconfig.json`? What do those compiler options mean?

**A:** Every option serves a purpose:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "noEmit": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true
  },
  "include": ["src/**/*"]
}
```

The key decisions:

- **`"module": "NodeNext"` + `"moduleResolution": "NodeNext"`** — Required for ESM in Node.js. This is what forces the `.js` import extensions throughout the codebase.
- **`"strict": true`** — Enables all strict type-checking flags: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, etc. This catches bugs at compile time.
- **`"noEmit": true`** — TypeScript is used **only for type checking**, not for compilation. The actual execution is handled by `tsx`, which transpiles on the fly.
- **`"noUncheckedIndexedAccess": true`** — This is a particularly careful setting. It makes indexed access (like `array[0]`) return `T | undefined` instead of just `T`, forcing you to handle the possibility that the element might not exist. This prevents a whole class of runtime errors.
- **`"esModuleInterop": true`** — Allows `import express from "express"` syntax with CommonJS packages instead of `import * as express from "express"`.

---

## 3. Type System

### Q: Walk me through the type definitions in `types/contact.ts`. Why are they structured this way?

**A:** The file defines five interfaces — each serving as a **data contract** for a specific operation:

```typescript
export interface Contact {
  id: number;
  name: string;
  address: string;
  type: "wallet" | "pda";
  createdAt: string;
}
```

**`Contact`** is the core entity. Notice:
- **`type: "wallet" | "pda"`** — This is a discriminated union. Every Solana address is either an "on-curve" wallet (has a private key) or an "off-curve" PDA (program-derived, no private key). The system auto-detects this when you add a contact.
- **`address`** is stored as a base58 string (the standard Solana address format, like `So11111111111111111111111111111111111111112`).
- **`createdAt`** is an ISO 8601 string — not a `Date` object — because it serializes cleanly to JSON.

```typescript
export interface CreateContactInput {
  name: string;
  address: string;
}
```

**`CreateContactInput`** is deliberately minimal — only what the user provides. The `id`, `type`, and `createdAt` are generated server-side. This follows the principle of never trusting client-supplied IDs or metadata.

```typescript
export interface UpdateContactInput {
  name: string;
}
```

**`UpdateContactInput`** only allows renaming. The address and type are immutable after creation. This is intentional — changing an address would break the identity of the contact.

```typescript
export interface DeriveAtaInput {
  mintAddress: string;
}

export interface VerifyOwnershipInput {
  address: string;
  message: string;
  signature: string;
}

export interface DerivePdaInput {
  programId: string;
  seeds: string[];
}
```

The remaining three interfaces define the payloads for the crypto endpoints. Each captures exactly the inputs needed for its respective Solana operation — nothing more, nothing less. This makes the API self-documenting.

---

## 4. Error Handling Strategy

### Q: How does error handling work across the application?

**A:** The project uses a **custom error hierarchy** combined with Express's global error handler. This is one of the cleanest patterns for API error handling:

```typescript
export class AppError extends Error {
  public readonly statusCode: number;

  constructor(message: string, statusCode: number) {
    super(message);
    this.statusCode = statusCode;
    Object.setPrototypeOf(this, AppError.prototype);
  }
}
```

`AppError` is the base class. The `Object.setPrototypeOf` call is crucial — when you extend built-in classes like `Error` in TypeScript, the prototype chain can break. This line fixes the `instanceof` check so that `err instanceof AppError` works correctly.

Then there are three specialized subclasses:

```typescript
export class BadRequestError extends AppError {    // 400
  constructor(message: string) { super(message, 400); }
}

export class NotFoundError extends AppError {       // 404
  constructor(message: string) { super(message, 404); }
}

export class ConflictError extends AppError {       // 409
  constructor(message: string) { super(message, 409); }
}
```

**Why this pattern?**

1. **Anywhere** in the codebase (services, validators, etc.) can `throw new BadRequestError("...")` without needing access to the Express `res` object.
2. The **global error handler** in `index.ts` catches all of them:

```typescript
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({ error: err.message });
    return;
  }
  console.error("Unhandled error:", err);
  res.status(500).json({ error: "Internal server error" });
});
```

3. If an unexpected error occurs (something that's **not** an `AppError`), it falls through to the generic 500 handler, which logs the full error but sends a sanitized message to the client. This prevents leaking stack traces or internal details.

---

### Q: Why not just use `res.status(400).json(...)` directly in controllers?

**A:** Because it couples HTTP concerns to business logic. Consider the `contactService.ts`:

```typescript
const existing = contacts.find((c) => c.address === normalizedAddress);
if (existing) {
  throw new ConflictError(`Contact with address ${normalizedAddress} already exists`);
}
```

The service doesn't know or care about HTTP. It throws a semantic error. The controller's `try/catch` forwards it to Express's `next(err)`, which hits the global handler. If you ever wanted to use this service outside of an HTTP context (CLI tool, WebSocket handler, etc.), the same errors would still make sense — you'd just handle them differently.

---

## 5. Validation Layer

### Q: What does the validation layer do, and why is it separate from the services?

**A:** The `utils/validation.ts` file contains three pure validation functions that are reused across the codebase. Keeping them in a utility module means they can be used from controllers, services, or anything else without circular dependencies.

---

### Q: Explain `validateSolanaAddress` in detail. What are all the checks?

**A:** This function performs a **three-layer validation** of a Solana address:

```typescript
export function validateSolanaAddress(address: string, fieldName: string = "address"): PublicKey {
  // Layer 1: Type check
  if (!address || typeof address !== "string") {
    throw new BadRequestError(`${fieldName} is required and must be a string`);
  }

  // Layer 2: Base58 decode + length check
  let decoded: Uint8Array;
  try {
    decoded = bs58.decode(address);
  } catch {
    throw new BadRequestError(`Invalid base58 encoding for ${fieldName}`);
  }
  if (decoded.length !== 32) {
    throw new BadRequestError(`${fieldName} must decode to exactly 32 bytes, got ${decoded.length}`);
  }

  // Layer 3: PublicKey construction
  try {
    return new PublicKey(address);
  } catch {
    throw new BadRequestError(`Invalid Solana address for ${fieldName}`);
  }
}
```

**Layer 1** — Ensures the input exists and is a string. Guards against `null`, `undefined`, numbers, etc.

**Layer 2** — Decodes the base58 string and verifies it's exactly 32 bytes. Solana public keys are 256-bit (32-byte) Ed25519 keys. A valid base58 string that decodes to 31 or 33 bytes is **not** a valid Solana address.

**Layer 3** — Constructs a `@solana/web3.js` `PublicKey` object, which performs its own internal validation. This is redundant with Layer 2 to some extent, but it's a defense-in-depth approach — if the Solana SDK adds additional validation in the future, this catches it.

The function **returns the `PublicKey`** object rather than just `true`/`false`, so callers can immediately use it without re-parsing. This avoids the "validate then parse" anti-pattern.

---

### Q: What about `validateSeeds`? Why does it check byte length?

**A:** PDA derivation on Solana has a hard constraint: **each seed must be ≤ 32 bytes**. This is enforced at the Solana runtime level — if you pass a longer seed, the transaction will fail.

```typescript
export function validateSeeds(seeds: unknown): string[] {
  if (!Array.isArray(seeds) || seeds.length === 0) {
    throw new BadRequestError("seeds must be a non-empty array of strings");
  }

  for (let i = 0; i < seeds.length; i++) {
    const seed = seeds[i];
    if (typeof seed !== "string") {
      throw new BadRequestError(`seeds[${i}] must be a string`);
    }

    const seedBytes = new TextEncoder().encode(seed);
    if (seedBytes.length > 32) {
      throw new BadRequestError(
        `seeds[${i}] exceeds 32 bytes (got ${seedBytes.length} bytes)`
      );
    }
  }

  return seeds as string[];
}
```

Notice it uses `TextEncoder().encode(seed)` for the byte-length check — not `seed.length`. This is important because JavaScript's `string.length` counts **UTF-16 code units**, not bytes. A string like `"café"` has 4 characters but 5 bytes in UTF-8. Since Solana uses raw byte buffers, the byte count is what matters.

---

## 6. Solana Service — Cryptographic Core

### Q: This is the heart of the Solana-specific logic. Let's start with `detectAddressType`. How does it differentiate wallets from PDAs?

**A:** This is one of the most elegant bits in the codebase:

```typescript
export function detectAddressType(pubkey: PublicKey): "wallet" | "pda" {
  try {
    const isOnCurve = PublicKey.isOnCurve(pubkey.toBytes());
    return isOnCurve ? "wallet" : "pda";
  } catch {
    return "pda";
  }
}
```

The distinction comes from **elliptic curve cryptography**. Solana uses the **Ed25519** curve. A regular wallet address is a point that lies **on** the Ed25519 curve — it was generated from a private key. A PDA is a point that's intentionally **off** the curve — it was derived deterministically and has no corresponding private key.

`PublicKey.isOnCurve()` checks whether the 32-byte public key decodes to a valid point on the Ed25519 curve. If it does → wallet. If it doesn't → PDA.

The `catch` block defaults to `"pda"` — if the curve check itself throws (malformed key bytes), it's safest to assume it's not a regular wallet.

---

### Q: Explain `deriveAta`. What is an Associated Token Account and why do we need to derive it?

**A:** On Solana, fungible tokens (SPL Tokens) aren't stored in your wallet directly. Instead, each wallet has a separate **Token Account** for each type of token it holds. The **Associated Token Account (ATA)** is the **canonical, deterministic** token account for a given wallet-mint pair.

```typescript
const TOKEN_PROGRAM_ID = new PublicKey("TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA");
const ASSOCIATED_TOKEN_PROGRAM_ID = new PublicKey("ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL");
```

These two addresses are **hard-coded Solana program IDs** — they never change. Every Solana developer knows them:
- `TokenkegQ...` is the **SPL Token Program** that manages token operations
- `ATokenGPv...` is the **Associated Token Account Program** that derives ATA addresses

```typescript
export function deriveAta(
  ownerAddress: string,
  mintAddress: string
): { ata: string; owner: string; mint: string } {
  const ownerPubkey = validateSolanaAddress(ownerAddress, "owner address");
  const mintPubkey = validateSolanaAddress(mintAddress, "mint address");

  const [ataPubkey] = PublicKey.findProgramAddressSync(
    [
      ownerPubkey.toBuffer(),
      TOKEN_PROGRAM_ID.toBuffer(),
      mintPubkey.toBuffer(),
    ],
    ASSOCIATED_TOKEN_PROGRAM_ID
  );

  return {
    ata: ataPubkey.toBase58(),
    owner: ownerPubkey.toBase58(),
    mint: mintPubkey.toBase58(),
  };
}
```

The derivation formula is: `PDA(seeds=[owner, TOKEN_PROGRAM_ID, mint], program=ASSOCIATED_TOKEN_PROGRAM_ID)`. This is **defined by the Solana specification** — every wallet, SDK, and explorer uses the exact same formula. Given the same owner and mint, you will always get the same ATA address.

`findProgramAddressSync` returns a tuple `[PublicKey, bump]`. The destructuring `const [ataPubkey]` grabs just the address — the bump seed (used internally to ensure the result is off-curve) is discarded here since the caller doesn't need it.

---

### Q: How does `verifySignature` work? What's the full flow of signature verification?

**A:** This implements **Ed25519 digital signature verification** — the same scheme Solana uses for transaction signing:

```typescript
export function verifySignature(
  address: string,
  message: string,
  signature: string
): boolean {
  // 1. Validate the address
  const pubkey = validateSolanaAddress(address, "address");

  // 2. Encode the message as UTF-8 bytes
  const messageBytes = new TextEncoder().encode(message);

  // 3. Decode the base58 signature
  let signatureBytes: Uint8Array;
  try {
    signatureBytes = bs58.decode(signature);
  } catch {
    throw new BadRequestError("Invalid base58 encoding for signature");
  }

  // 4. Validate signature length (Ed25519 signatures are always 64 bytes)
  if (signatureBytes.length !== 64) {
    throw new BadRequestError(
      `Signature must be 64 bytes, got ${signatureBytes.length}`
    );
  }

  // 5. Verify using tweetnacl
  const publicKeyBytes = pubkey.toBytes();
  return nacl.sign.detached.verify(messageBytes, signatureBytes, publicKeyBytes);
}
```

**The idea:** A user signs a message with their wallet's private key off-chain. They send the public key (address), the original message, and the signature to this endpoint. The server verifies that only the holder of the corresponding private key could have produced that signature.

**`nacl.sign.detached.verify`** is the core — it's a pure cryptographic operation:
- Takes the message bytes, 64-byte signature, and 32-byte public key
- Returns `true` if the signature is valid, `false` otherwise
- "Detached" means the signature doesn't include the message (unlike "attached" signatures)

This is commonly used for **"Sign-In with Solana"** or **proof of ownership** — proving you control a wallet without making an on-chain transaction.

---

### Q: And `derivePda`? How is it different from ATA derivation?

**A:** ATA derivation is a **specific case** of PDA derivation with fixed seeds and a fixed program. `derivePda` is the **general case** — derive a PDA for **any** program with **any** seeds:

```typescript
export function derivePda(
  programIdStr: string,
  seeds: string[]
): { pda: string; bump: number } {
  const programId = validateSolanaAddress(programIdStr, "programId");
  const validatedSeeds = validateSeeds(seeds);

  const seedBuffers = validatedSeeds.map(
    (seed) => Buffer.from(new TextEncoder().encode(seed))
  );

  const [pdaPubkey, bump] = PublicKey.findProgramAddressSync(
    seedBuffers,
    programId
  );

  return {
    pda: pdaPubkey.toBase58(),
    bump,
  };
}
```

**Unlike ATA derivation, this returns the `bump` seed.** The bump is a single byte (0–255) that `findProgramAddressSync` iterates through to find an off-curve address. Programs often need this bump to **re-derive the PDA on-chain** for verification, so it's valuable to return it.

**The seed conversion:** `Buffer.from(new TextEncoder().encode(seed))` converts each string seed to a byte buffer via UTF-8 encoding. This is important — `"hello"` becomes `[104, 101, 108, 108, 111]`, five bytes. The same conversion must happen on-chain for the PDA to match.

---

## 7. Contact Service — Business Logic

### Q: How does the in-memory storage work? What are the trade-offs?

**A:** The storage is intentionally simple — a module-level array and a counter:

```typescript
const contacts: Contact[] = [];
let nextId = 1;
```

**Trade-offs:**

| Pros | Cons |
|------|------|
| Zero setup — no database needed | Data lost on server restart |
| Extremely fast — array operations | No persistence across sessions |
| Perfect for prototyping/contests | Doesn't scale beyond a single process |
| Easy to test — predictable state | No concurrent access safety |

The `nextId` counter is a simple auto-increment — every new contact gets the next integer. This is safe in a single-threaded Node.js environment but would need an atomic counter or database sequence in production.

---

### Q: Walk through `createContact`. What happens step by step?

**A:**

```typescript
export function createContact(input: CreateContactInput): Contact {
  // Step 1: Validate the name — must be non-empty string
  const name = validateRequiredString(input.name, "name");

  // Step 2: Validate the address — must be non-empty string
  const addressStr = validateRequiredString(input.address, "address");

  // Step 3: Validate it's a real Solana address (base58, 32 bytes, valid PublicKey)
  const pubkey = validateSolanaAddress(addressStr, "address");

  // Step 4: Normalize the address to its canonical base58 form
  const normalizedAddress = pubkey.toBase58();

  // Step 5: Check for duplicate addresses
  const existing = contacts.find((c) => c.address === normalizedAddress);
  if (existing) {
    throw new ConflictError(`Contact with address ${normalizedAddress} already exists`);
  }

  // Step 6: Auto-detect whether the address is a wallet or PDA
  const addressType = detectAddressType(pubkey);

  // Step 7: Build the contact object
  const contact: Contact = {
    id: nextId++,
    name,
    address: normalizedAddress,
    type: addressType,
    createdAt: new Date().toISOString(),
  };

  // Step 8: Store and return
  contacts.push(contact);
  return contact;
}
```

**Step 4 is subtle but important:** `pubkey.toBase58()` normalizes the address. Base58 can have different valid representations of the same bytes in edge cases — normalizing ensures that duplicate checks (Step 5) work correctly.

**Step 6 is automatic:** The user never specifies the type — the system determines it via the Ed25519 curve check. This is a nice UX touch — users don't need to know the difference.

---

### Q: How does `getContacts` handle filtering and sorting?

**A:**

```typescript
export function getContacts(typeFilter?: string): Contact[] {
  let result = [...contacts]; // Shallow copy to avoid mutation

  if (typeFilter) {
    if (typeFilter !== "wallet" && typeFilter !== "pda") {
      throw new BadRequestError('type must be "wallet" or "pda"');
    }
    result = result.filter((c) => c.type === typeFilter);
  }

  result.sort((a, b) => a.id - b.id); // Ascending by ID
  return result;
}
```

Key points:
- **`[...contacts]`** — Creates a shallow copy. This prevents the sort from mutating the original array.
- **Type validation** — Only `"wallet"` and `"pda"` are accepted. Any other value is a `BadRequestError`.
- **Sort by ID ascending** — Ensures consistent ordering regardless of internal array order.

---

### Q: What about `deleteContact`? How does `splice` work here?

**A:**

```typescript
export function deleteContact(id: number): Contact {
  const index = contacts.findIndex((c) => c.id === id);
  if (index === -1) {
    throw new NotFoundError("Contact not found");
  }

  const [deleted] = contacts.splice(index, 1);
  return deleted as Contact;
}
```

`splice(index, 1)` removes one element at the given index **in-place** and returns an array of removed elements. The destructuring `const [deleted]` extracts the single removed contact. The `as Contact` assertion is needed because TypeScript's strict mode with `noUncheckedIndexedAccess` makes the type `Contact | undefined`, but we know it exists since `splice` returns exactly the elements it removed.

---

## 8. Controllers — Request Handling

### Q: What's the role of the controllers? How do they differ from services?

**A:** Controllers are the **HTTP boundary** — they translate between Express's `Request`/`Response` objects and the service layer's pure TypeScript functions. They handle:

1. **Parsing** request parameters, query strings, and bodies
2. **Calling** the appropriate service function
3. **Formatting** the response (status codes, JSON)
4. **Error forwarding** via `next(err)`

They **never** contain business logic or data validation (beyond parsing route params).

---

### Q: Let's look at `handleCreateContact`. Why is it `async` if `createContact` is synchronous?

**A:**

```typescript
export async function handleCreateContact(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const input = req.body as CreateContactInput;
    const contact = createContact(input);
    res.status(201).json(contact);
  } catch (err) {
    next(err);
  }
}
```

The `async` keyword is a **forward-compatibility and Express convention** choice. Even though `createContact` is synchronous today, marking the handler as `async` means:
1. Any thrown errors inside the function are automatically caught by Express 5's async error handling
2. If the service becomes asynchronous in the future (e.g., database calls), the handler doesn't need to change
3. The `Promise<void>` return type satisfies Express 5's type expectations

The `try/catch` + `next(err)` pattern is the controller's **entire error strategy** — it delegates everything to the global error handler.

---

### Q: How does `handleGetContactById` parse the route parameter?

**A:**

```typescript
export async function handleGetContactById(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const id = parseInt(String(req.params.id), 10);
    if (isNaN(id)) {
      res.status(400).json({ error: "id must be a number" });
      return;
    }
    const contact = getContactById(id);
    res.json(contact);
  } catch (err) {
    next(err);
  }
}
```

Route parameters in Express are always strings (e.g., `/contacts/42` gives `req.params.id === "42"`). The controller:
1. Wraps it in `String()` as a safety guard (in case it's somehow `undefined`)
2. Parses with `parseInt(..., 10)` — the radix `10` is explicit to avoid octal/hex surprises
3. Validates with `isNaN()` — catches inputs like `/contacts/abc`

Note that the `NaN` check is handled **directly** in the controller (not thrown as an error) because this is a **parsing concern**, not a business logic concern. It's the controller's job to translate HTTP inputs into valid function arguments.

---

### Q: What about `handleDeriveAta`? It combines contact lookup with crypto operations.

**A:**

```typescript
export async function handleDeriveAta(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const id = parseInt(String(req.params.id), 10);
    if (isNaN(id)) {
      res.status(400).json({ error: "id must be a number" });
      return;
    }

    const { mintAddress } = req.body as DeriveAtaInput;
    validateRequiredString(mintAddress, "mintAddress");

    const contact = getContactById(id);
    const result = deriveAta(contact.address, mintAddress);
    res.json(result);
  } catch (err) {
    next(err);
  }
}
```

This is a great example of **composition** — the controller orchestrates two services:
1. First it fetches the contact by ID (contact service)
2. Then it derives the ATA using the contact's address and the provided mint (Solana service)

If the contact doesn't exist, `getContactById` throws a `NotFoundError`. If the mint address is invalid, `deriveAta` throws a `BadRequestError`. Either way, `next(err)` catches it.

---

### Q: How does `cryptoController.ts` handle the verify-ownership and derive-pda endpoints?

**A:** The crypto controller handles two standalone endpoints that aren't tied to any specific contact:

```typescript
export async function handleVerifyOwnership(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const { address, message, signature } = req.body as VerifyOwnershipInput;

    validateRequiredString(address, "address");
    validateRequiredString(message, "message");
    validateRequiredString(signature, "signature");

    const valid = verifySignature(address, message, signature);
    res.json({ valid });
  } catch (err) {
    next(err);
  }
}
```

Notice: it validates all three fields are present **before** calling the service. This is a **fail-fast** approach — if `message` is missing, it doesn't waste time validating the address.

```typescript
export async function handleDerivePda(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  try {
    const { programId, seeds } = req.body as DerivePdaInput;

    validateRequiredString(programId, "programId");

    if (!seeds || !Array.isArray(seeds)) {
      res.status(400).json({ error: "seeds must be a non-empty array of strings" });
      return;
    }

    const result = derivePda(programId, seeds);
    res.json(result);
  } catch (err) {
    next(err);
  }
}
```

The seeds check in the controller is **preliminary** — it ensures the field exists and is an array before passing to `validateSeeds()` (inside `derivePda`), which does the thorough per-element validation. This avoids cryptic errors from passing garbage to the validator.

---

## 9. Routing Layer

### Q: How are the routes organized, and what's the full API surface?

**A:** Two route files define the complete API:

**`routes/contacts.ts`** — Mounted at `/api/contacts`:

```
POST   /api/contacts              → Create a new contact
GET    /api/contacts              → List all contacts (optional ?type=wallet|pda)
GET    /api/contacts/:id          → Get a single contact by ID
PUT    /api/contacts/:id          → Update a contact's name
DELETE /api/contacts/:id          → Delete a contact
POST   /api/contacts/:id/derive-ata → Derive ATA for a contact's address
```

**`routes/crypto.ts`** — Mounted at `/api`:

```
POST   /api/verify-ownership      → Verify an Ed25519 signature
POST   /api/derive-pda            → Derive a PDA from program ID + seeds
```

The routing is minimal — just mapping verbs and paths to handlers. No middleware, no guards, no auth. The routers use Express's `Router()` factory, which creates modular route groups that are mounted by the main app.

---

### Q: Why are the crypto routes mounted at `/api` instead of `/api/crypto`?

**A:** This is likely a design choice for **URL clarity**. `/api/verify-ownership` reads more naturally than `/api/crypto/verify-ownership`. The crypto operations are top-level API features, not sub-resources of a "crypto" entity. It also keeps URLs shorter and more RESTful — the verb is in the function name, not the URL.

---

## 10. Application Entry Point

### Q: Walk me through `index.ts`. What happens when the server starts?

**A:**

```typescript
import express from "express";
import type { Request, Response, NextFunction } from "express";
import contactsRouter from "./routes/contacts.js";
import cryptoRouter from "./routes/crypto.js";
import { AppError } from "./utils/errors.js";

const app = express();
const PORT = 3000;
```

**Step 1: Imports and initialization.** The `express()` factory creates the application instance. Port is hardcoded to `3000`.

```typescript
app.use(express.json());
```

**Step 2: Middleware.** `express.json()` is the JSON body parser — it reads incoming `Content-Type: application/json` bodies and populates `req.body`. Without this, `req.body` would be `undefined` for POST/PUT requests.

```typescript
app.use("/api/contacts", contactsRouter);
app.use("/api", cryptoRouter);
```

**Step 3: Route mounting.** Each router is mounted at its base path. Express concatenates the mount path with the router's internal paths — so `router.post("/")` inside `contactsRouter` becomes `POST /api/contacts/`.

```typescript
app.get("/health", (_req: Request, res: Response) => {
  res.json({ status: "ok", timestamp: new Date().toISOString() });
});
```

**Step 4: Health check.** A simple endpoint that returns `{"status": "ok"}` with a timestamp. This is standard practice for load balancers, monitoring systems, and container orchestrators (like Kubernetes) to verify the service is alive.

```typescript
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({ error: err.message });
    return;
  }
  console.error("Unhandled error:", err);
  res.status(500).json({ error: "Internal server error" });
});
```

**Step 5: Global error handler.** Express recognizes this as an error handler because it has **four parameters** (including `err`). All errors forwarded via `next(err)` from controllers land here. It checks if the error is one of ours (`AppError`) and formats it with the appropriate status code, or returns a generic 500 for unexpected errors.

```typescript
app.listen(PORT, () => {
  console.log(`🚀 Solana Address Book API running on http://localhost:${PORT}`);
  console.log(`📋 Health check: http://localhost:${PORT}/health`);
});
```

**Step 6: Start listening.** The callback fires once the server is bound to the port and ready to accept connections.

---

## 11. Data Flow — End to End

### Q: Trace a complete request from HTTP to response. Let's say someone creates a contact.

**A:** Here's the full journey of `POST /api/contacts` with body `{"name": "My Wallet", "address": "So11111111111111111111111111111111111111112"}`:

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. CLIENT sends POST /api/contacts                                  │
│    Body: { "name": "My Wallet", "address": "So111..." }             │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. EXPRESS.JSON() MIDDLEWARE                                        │
│    Parses JSON body → populates req.body                            │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. ROUTER matches POST /api/contacts → handleCreateContact          │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. CONTROLLER (contactsController.ts)                               │
│    - Extracts req.body as CreateContactInput                        │
│    - Calls createContact(input)                                     │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. SERVICE (contactService.ts)                                      │
│    - validateRequiredString(name)     → "My Wallet"                 │
│    - validateRequiredString(address)  → "So111..."                  │
│    - validateSolanaAddress(address)   → PublicKey instance           │
│    - Normalize address via .toBase58()                               │
│    - Check duplicates in contacts[]                                  │
│    - detectAddressType(pubkey)        → "wallet" or "pda"           │
│    - Build Contact object with id, name, address, type, createdAt   │
│    - Push to contacts[], return Contact                              │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. CONTROLLER sends res.status(201).json(contact)                   │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 7. CLIENT receives 201 Created                                      │
│    { "id": 1, "name": "My Wallet", "address": "So111...",          │
│      "type": "wallet", "createdAt": "2026-04-03T..." }             │
└─────────────────────────────────────────────────────────────────────┘
```

**If any step fails**, the error path is:

```
Service throws BadRequestError / ConflictError / NotFoundError
  → Controller's catch block calls next(err)
    → Global error handler checks instanceof AppError
      → Sends { "error": "message" } with correct HTTP status code
```

---

## 12. Security Considerations

### Q: What security measures does this codebase implement?

**A:** Several, though it's primarily designed as a contest/prototype:

1. **Input validation at every boundary** — Addresses are validated for base58 encoding, byte length, and PublicKey construction. Seeds are validated for byte length. Strings are checked for presence and type.

2. **Error sanitization** — Unexpected errors return a generic `"Internal server error"` message; only known `AppError` messages are exposed to clients. Stack traces are logged server-side only.

3. **No raw user input in responses** — Addresses are normalized via `toBase58()` before storage, preventing homoglyph or encoding attacks.

4. **Duplicate prevention** — The `ConflictError` on duplicate addresses prevents data corruption.

5. **Cryptographic verification** — The `verifySignature` endpoint uses the battle-tested `tweetnacl` library, not a custom implementation.

### Q: What's NOT protected?

**A:** Since this is a contest/prototype project:

- **No authentication or authorization** — Anyone can create, read, update, or delete contacts
- **No rate limiting** — The API could be abused with rapid requests
- **No CORS configuration** — Browser-based clients from other origins would be blocked by default
- **No HTTPS** — Data travels in plaintext (in production, you'd use a reverse proxy like Nginx)
- **No persistence** — All data is lost on restart (by design for this use case)

---

> **End of Deep Dive** — This document covers every file, every function, and every design decision in the Solana Address Book API. Each component is built with clear separation of concerns, making the codebase easy to extend, test, and reason about.
