# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Go SDK and reference implementation for Lighter exchange transaction signing and hashing. The project compiles to shared libraries (C-shared buildmode) for cross-platform consumption:
- macOS: `.dylib` (arm64)
- Linux: `.so` (amd64, arm64)
- Windows: `.dll` (amd64)

The SDK handles cryptographic signing of Lighter exchange transactions using Schnorr signatures over Poseidon2 hashing (Goldilocks field). It provides both a Go SDK and C-compatible exports for integration with other languages.

## Build Commands

This project uses `just` as the task runner. All build commands are defined in the `justfile`.

### Local Builds
```bash
# macOS (arm64)
just build-darwin-local

# Linux (platform-specific)
just build-linux-local

# Windows (requires msys2/mingw)
just build-windows-local
```

### Docker Builds (Cross-platform)
```bash
# Linux amd64
just build-linux-amd64-docker

# Linux arm64
just build-linux-arm64-docker

# Windows amd64 (cross-compile with mingw)
just build-windows-amd64-docker
```

All builds:
1. Run `go mod vendor` first
2. Use `-buildmode=c-shared` to create shared libraries
3. Output to `./build/` directory
4. Compile `sharedlib/sharedlib.go` as the entry point

### Development Commands
```bash
# Vendor dependencies (required before building)
go mod vendor

# View available build tasks
just --list
```

## Architecture

### High-Level Structure

```
├── sharedlib/        # C-compatible exports (CGO), main entry point for shared lib
├── client/           # TxClient and HTTP client for API interactions
│   └── http/         # HTTP client implementation
├── signer/           # Cryptographic key management and signing
├── types/            # High-level transaction request types
│   └── txtypes/      # Low-level transaction types with hashing/validation
└── vendor/           # Vendored dependencies (after go mod vendor)
```

### Key Components

#### 1. `sharedlib/sharedlib.go` - C Export Layer
- Entry point for shared library compilation
- Exports all transaction signing functions with C-compatible signatures
- Manages global state: `defaultTxClient`, `allTxClients` map
- Implements async job queue for non-blocking operations (`SignCreateOrderAsync`, `GetJobStatus`)
- Uses CGO to convert between Go and C types
- All exported functions follow pattern: accept C types, defer error handling, return `C.StrOrErr` or `C.ApiKeyResponse`

**Multi-client Management:**
- `CreateClient()` initializes a client for a specific (account, apiKey) pair
- Clients stored in `allTxClients[accountIndex][apiKeyIndex]`
- Default values (apiKeyIndex=255, accountIndex=-1) use `defaultTxClient`
- Each client maintains its own keyManager and chainId

**Async Job Infrastructure:**
- `jobQueue` stores results with auto-cleanup (1 minute TTL)
- Jobs identified by client-provided string IDs
- Pattern: Call `*Async()` function, poll with `GetJobStatus(jobId)`

#### 2. `client/` - Transaction Client
**`client/tx_client.go`:**
- `TxClient` struct links to specific (accountIndex, apiKeyIndex) pair
- Manages `keyManager`, `chainId`, and optional `apiClient` (for nonce fetching)
- `FullFillDefaultOps()` fills missing TransactOpts (nonce, expiry, account/API key indices)
- Default transaction expiry: 10 minutes from creation
- If nonce=-1 and apiClient exists, fetches nonce via HTTP

**`client/tx_get.go`:**
- Methods like `GetCreateOrderTransaction()`, `GetWithdrawTransaction()`, etc.
- Each method: validates ops → constructs tx → signs → returns signed transaction
- All transactions validated using Schnorr signature verification

**`client/http/`:**
- `MinimalHTTPClient` interface for API calls
- Used for: `GetNextNonce()`, `GetApiKey()` (client verification)
- Custom HTTP transport with connection pooling (1000 max, 100 idle, 10s timeout)

#### 3. `signer/key_manager.go` - Cryptography
- `KeyManager` interface: Sign, PubKey, PubKeyBytes, PrvKeyBytes
- Private keys are 40-byte Goldilocks quintic extension scalars
- Signing: Uses Schnorr signatures over Poseidon2 hash function
- `NewKeyManager()` requires exactly 40 bytes (not hex-encoded)
- Public keys derived via `schnorr.SchnorrPkFromSk()`

#### 4. `types/` - Transaction Types
**`types/tx_request.go`:**
- High-level request types (e.g., `CreateOrderTxReq`, `WithdrawTxReq`)
- `TransactOpts`: Common options (nonce, expiry, account/API key indices)
- `Construct*Tx()` functions: Convert request → txtypes → sign → return signed tx
- `Convert*Tx()` functions: Transform high-level types to low-level txtypes

**`types/txtypes/`:**
- Low-level transaction structs (e.g., `L2CreateOrderTxInfo`)
- Each type implements:
  - `Validate()` - Field validation
  - `Hash(chainId)` - Poseidon2 hashing for signing
  - `GetL1SignatureBody()` - For transactions requiring L1 signatures (Transfer, ChangePubKey)
- Constants for order types, time-in-force, margin modes, etc.

### Transaction Signing Flow

1. **Caller** → Creates high-level request (e.g., `CreateOrderTxReq`)
2. **sharedlib** → Calls `GetCreateOrderTransaction()` on appropriate TxClient
3. **TxClient** → Calls `FullFillDefaultOps()` to fill nonce/expiry
4. **types** → `ConstructCreateOrderTx()` converts request → `L2CreateOrderTxInfo`
5. **txtypes** → `Validate()` checks fields, `Hash(chainId)` generates message hash
6. **signer** → `Sign(msgHash)` produces Schnorr signature
7. **Return** → Signed transaction with signature and hash attached

### Nonce Management

The SDK supports three nonce handling modes:

1. **Manual (recommended for production)**: Caller manages nonces locally to avoid HTTP latency
   - Pass explicit nonce value (≥0)
   - Each API key has independent nonce sequence

2. **Auto-fetch (default)**: Pass nonce=-1, SDK fetches via HTTP
   - Requires `apiClient` to be set (non-nil URL in `CreateClient`)
   - Adds latency but simpler for testing

3. **Default client usage**: Pass (apiKeyIndex=255, accountIndex=-1)
   - Uses `defaultTxClient` (last created client)

### Auth Tokens

- Valid for 8 hours maximum
- Bound to specific API key (changing key invalidates all tokens)
- Format: `{deadline_unix}:{accountIndex}:{apiKeyIndex}:{signature}`
- `CreateAuthToken(deadline=0)` defaults to 7 hours from now
- Can pre-generate tokens with future deadlines (start valid in future)

### Async Operations

**Pattern:**
```go
// Client calls async function with job ID
SignCreateOrderAsync(...params..., jobId)

// Poll for completion
status := GetJobStatus(jobId)
// Returns: {"completed": true, "result": "{...}", "error": ""}
```

**Cleanup:**
- Jobs auto-delete after 1 minute of completion
- Manual cleanup: `CleanupOldJobs()`

## Important Implementation Notes

1. **Private Key Format**:
   - TxClient expects hex-encoded string (with or without "0x")
   - KeyManager requires raw 40-byte array
   - Conversion handled in `NewTxClient()`

2. **Account/API Key Indexing**:
   - accountIndex must be >0 (1-based)
   - apiKeyIndex is 0-255 (uint8)
   - All API keys are equal in permissions
   - Each API key has independent nonce sequence

3. **Transaction Expiry**:
   - Default: 10 minutes (`DefaultExpireTime` in tx_client.go)
   - Order expiry default: 28 days (in sharedlib.go)
   - Expiry can be overridden via `TransactOpts.ExpiredAt`

4. **Memo Fields** (Transfer transactions):
   - Must be exactly 32 bytes
   - No validation of content, only length

5. **Error Handling in CGO**:
   - All exports use `defer` with panic recovery
   - Errors returned as C strings via `wrapErr()`
   - Success returns nil error pointer

6. **Vendor Directory**:
   - Dependencies must be vendored before building (`go mod vendor`)
   - Critical deps: `poseidon_crypto`, `go-ethereum`

## Cryptographic Details

- **Curve**: Goldilocks quintic extension field (ECgFp5)
- **Hash Function**: Poseidon2 (ZK-friendly hash)
- **Signature Scheme**: Schnorr signatures
- **Key Size**: 40 bytes (320 bits) for private keys and public keys
- **API Key Generation**: `GenerateAPIKey(seed)` samples random scalar (optional seed)

## Related Projects

- **Python SDK**: [github.com/elliottech/lighter-python](https://github.com/elliottech/lighter-python) - Full HTTP/WebSocket support with examples
- **Poseidon Crypto**: [github.com/elliottech/poseidon_crypto](https://github.com/elliottech/poseidon_crypto) - Cryptographic primitives

## Gotchas

- Windows builds require msys2/mingw for cross-compilation
- Docker darwin builds likely don't work (commented out in justfile)
- No automatic retry on HTTP failures - caller must handle
- `CheckClient()` validates API key matches server, but doesn't verify account ownership
- Async jobs have 1-minute retention; lost if not polled in time
- `go mod vendor` is required before every build command
