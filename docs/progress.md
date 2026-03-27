# BlocRoc Development Progress

This document captures everything built so far, decisions made, known issues, and what comes next. Written so a future AI chat (or human contributor) can pick up exactly where we left off.

---

## Commit History

| Commit | Description |
|---|---|
| `abccbfa` | Scaffold full monorepo: roc-chain (node, runtime, 4 pallets), roc-frontend, roc-scanner, roc-indexer, docs, scripts |
| `f573508` | Add system architecture diagram to docs |
| `fc27999` | Add plain-English project explainer (docs/explainer.md) |
| `d8bed05` | **Prompt 2.1** — Ticket pallet data structures and storage items |
| `c5a9a00` | **Prompt 2.2** — Ticket pallet extrinsics, tests, chain spec, network scripts |

GitHub: `https://github.com/jeremymitaux/BlocRoc` (main branch)

---

## What Has Been Built

### 1. Monorepo Structure

```
BlocRoc/
├── roc-chain/
│   ├── node/           Substrate node binary (Aura + GRANDPA consensus)
│   ├── runtime/        WASM runtime composing all pallets
│   └── pallets/
│       ├── ticket/     ** IMPLEMENTED — data structures + 5 extrinsics **
│       ├── event/      Stubbed (compiles, no logic yet)
│       ├── marketplace/ Stubbed (compiles, no logic yet)
│       └── scanner/    Stubbed (compiles, no logic yet)
├── roc-frontend/       Next.js app (scaffolded, not implemented)
├── roc-scanner/        React Native app (scaffolded, not implemented)
├── roc-indexer/        SubQuery indexer (scaffolded, not implemented)
├── docs/               Architecture docs, API spec, explainer
├── scripts/            setup.sh, run-tests.sh, demo.sh, start-network.sh, stop-network.sh
└── Cargo.toml          Workspace root
```

### 2. Ticket Pallet (fully implemented)

**Location:** `roc-chain/pallets/ticket/src/`

**Data Structures (Prompt 2.1):**
- `EventDetails<T>` — event record: name, venue, date, capacity, ticket_price, resale_cap_percent, tickets_sold, creator, is_active
- `TicketDetails<T>` — ticket record: event_id, tier, seat, original_price, current_price, owner, is_used, is_listed_for_resale
- `TransferRejectedReason` — enum for rejection reasons in events

**Storage Items:**
- `NextEventId` — auto-increment counter (StorageValue<u64>)
- `NextTicketId` — auto-increment counter (StorageValue<u64>)
- `Events` — StorageMap<EventId, EventDetails>
- `Tickets` — StorageMap<TicketId, TicketDetails>
- `TicketsByOwner` — StorageDoubleMap<AccountId, TicketId, ()> (reverse index)
- `TicketsByEvent` — StorageDoubleMap<EventId, TicketId, ()> (reverse index)

**5 Extrinsics (Prompt 2.2):**

| # | Extrinsic | What it does |
|---|---|---|
| 0 | `create_event` | Any signed account creates an event, stores EventDetails, emits EventCreated |
| 1 | `mint_tickets` | Event creator mints a batch of tickets (checks capacity). Tickets owned by creator initially |
| 2 | `purchase_ticket` | Buyer pays ticket_price to creator via `T::Currency::transfer`. Primary sale only (owner must be creator) |
| 3 | `transfer_ticket` | Secondary market transfer. Enforces resale cap: `price <= original_price * resale_cap_percent / 100`. Arithmetic done in u128 via `SaturatedConversion` to avoid `From<u64>` bound issues |
| 4 | `validate_ticket` | Scanner marks ticket as used (permanent). Any signed origin (production: restrict via EnsureOrigin) |

**Config trait:**
```rust
type RuntimeEvent: From<Event<Self>> + IsType<...>;
type Currency: Currency<Self::AccountId>;
type MaxStringLength: Get<u32>;  // bounds all BoundedVec<u8> fields
type WeightInfo: WeightInfo;
```

**Tests: 39 passing** (`cargo test -p pallet-ticket`)
- All happy paths for all 5 extrinsics
- All error branches (not found, not owner, not creator, event inactive, sold out, already used, resale cap exceeded, insufficient funds)
- Encode/decode round-trip tests (compare encoded bytes to avoid PartialEq issues with parameter_types structs)
- Full lifecycle integration test: create -> mint -> buy -> transfer -> validate -> reject double-use
- BoundedVec boundary tests

**Key implementation decisions:**
- Resale cap math uses `sp_runtime::SaturatedConversion` to convert `BalanceOf<T>` to `u128`, do integer math, then compare. This avoids needing a `From<u64>` bound on the Balance type.
- `tickets_sold` tracks purchases (not mints). Capacity check in `mint_tickets` uses `capacity - tickets_sold` as remaining.
- `purchase_ticket` is primary sale only — owner must be the event creator. Secondary market uses `transfer_ticket`.
- Currency transfers use `ExistenceRequirement::KeepAlive` and map errors to `Error::<T>::InsufficientFunds`.

### 3. Runtime Integration

**Location:** `roc-chain/runtime/src/lib.rs`

The ticket pallet is wired into the runtime:
```rust
impl pallet_ticket::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type Currency = Balances;
    type MaxStringLength = ConstU32<128>;
    type WeightInfo = pallet_ticket::weights::SubstrateWeight<Runtime>;
}
```

It's included in `construct_runtime!` as `Ticket: pallet_ticket`.

The runtime uses **Aura (sr25519) for block production** and **GRANDPA (ed25519) for finality**. NOT BABE.

### 4. Chain Specification

**Location:** `roc-chain/node/src/chain_spec.rs`

Three chain specs:
- `dev` — single validator (Alice), for local development
- `local` — two validators (Alice, Bob), for local multi-node
- `blocroc` — **4-validator testnet** named after music venues:

| Validator | Venue Name | Authority Seed |
|---|---|---|
| 1 | The Roxy | alice |
| 2 | Red Rocks | bob |
| 3 | House of Blues | charlie |
| 4 | Local Dive Bar | dave |

All 6 dev accounts (Alice–Ferdie) are endowed with 1M ROC tokens (18 decimals). Alice is sudo. Protocol ID: `blocroc`.

`command.rs` loads `"blocroc"` as a chain ID option.

### 5. Network Scripts

- **`scripts/start-network.sh`** — Purges chain data, generates raw chain spec JSON, starts 4 validator nodes as background processes on ports 30333–30336 (P2P) and 9944–9947 (RPC). Injects Aura (sr25519) and GRANDPA (ed25519) keys via `author_insertKey` RPC. Node 1 uses a deterministic node-key for bootnode discovery.
- **`scripts/stop-network.sh`** — Reads PID file and sends SIGTERM to all nodes (SIGKILL after 2s if needed). Falls back to `pgrep` if PID file missing.

### 6. Workspace Dependencies

**Location:** `Cargo.toml` (workspace root)

All Substrate/Polkadot-SDK crates use git dependencies:
```toml
git = "https://github.com/paritytech/polkadot-sdk"
tag = "polkadot-stable2409"
```

This was switched from crates.io version numbers (which had version mismatch issues) to git tags in an earlier session. The original template used `polkadot-stable2412` but that had a broken `fflonk` transitive dependency, so we downgraded to `polkadot-stable2409`.

---

## Known Issues / Blockers

### CRITICAL: Full node `cargo build --release` does not compile

The pallet tests compile and pass (`cargo test -p pallet-ticket`), but building the full node binary fails due to Rust toolchain vs. polkadot-stable2409 ecosystem incompatibility:

**Problem 1 — Modern Rust (1.85+):** `sc-network` has a `#[derive(Decode)]` on an enum with duplicate variant indexes (both `Consensus` and `RemoteCallResponse` have index 6). Newer Rust's stricter const evaluation catches this as `error[E0080]`. This was fixed in `polkadot-stable2409-5`.

**Problem 2 — polkadot-stable2409-5+:** The patch tags (-5 through -12) change runtime APIs: `RuntimeVersion` drops `system_version`, `sp_genesis_builder` renames functions, `pallet_aura::authorities()` changes signature, `derive_impl(SolochainDefaultConfig)` changes `AccountData` defaults. The runtime code would need updating to match.

**Problem 3 — Older Rust (1.81–1.84):** Newer crate versions in the dependency graph (`clap_lex 1.1.0`, `base64ct 1.8.3`) require Rust edition 2024, which wasn't stabilized until Rust 1.85.

**Problem 4 — WASM build:** Even when native compilation works, the WASM runtime build fails because newer `serde` versions (1.0.220+) introduce a `serde_core` internal crate that pulls `std` into the `no_std` WASM target, conflicting with `sp-io`'s `panic_impl` lang item.

**Resolution path (not yet attempted):**
1. Upgrade to `polkadot-stable2409-5` (or later) and update `runtime/src/lib.rs` to match the new APIs. Specifically:
   - Remove `system_version` from `RuntimeVersion`
   - Update `sp_genesis_builder` impl (function signatures changed)
   - Fix `Aura::authorities()` call
   - Add explicit `type AccountData = pallet_balances::AccountData<u128>` to `frame_system::Config` to override the derive_impl default
   - Address `create_runtime_str!` import change
2. Verify the serde/WASM issue is resolved in -5+ (it may be, since the SDK patches address WASM compatibility)
3. Pin `rust-toolchain.toml` to whatever Rust version works with the chosen tag

### Other Pre-existing Issues (from audit in earlier session)

These were identified but the user said "lets not fix these quite yet":
1. `service.rs` has a `todo!()` macro that will panic at runtime (network config)
2. `sc_network::config::NonDefaultSetConfig` moved in polkadot-stable2409
3. Pallet name collision: `Event` (pallet-event) vs Substrate's `Event` type
4. Stubbed pallets (event, marketplace, scanner) have placeholder code only
5. Runtime `Cargo.toml` had `pallet-transaction-payment/runtime-benchmarks` feature that doesn't exist in stable2409 (already removed)

---

## What Comes Next

The user has been following a prompt-based development plan. Completed prompts:

- **Prompt 2.1** — Ticket pallet data structures and storage (done)
- **Prompt 2.2** — Ticket pallet extrinsics and tests (done)

Likely next steps:
1. **Fix the full node build** — upgrade to polkadot-stable2409-5+, update runtime APIs, verify WASM builds
2. **Prompt 3.x** — Implement the remaining pallets (event, marketplace, scanner) with real logic
3. **Frontend** — Next.js dashboard and marketplace
4. **Scanner app** — React Native mobile scanner
5. **Indexer** — SubQuery mapping functions
6. **CI** — GitHub Actions for build + test

---

## File Quick Reference

| File | What's in it |
|---|---|
| `roc-chain/pallets/ticket/src/lib.rs` | Pallet impl: config, storage, events, errors, 5 extrinsics |
| `roc-chain/pallets/ticket/src/mock.rs` | Test runtime with frame_system + pallet_balances + pallet_ticket |
| `roc-chain/pallets/ticket/src/tests.rs` | 39 unit tests covering all extrinsics |
| `roc-chain/pallets/ticket/src/weights.rs` | Placeholder weight functions (need benchmarking before mainnet) |
| `roc-chain/runtime/src/lib.rs` | Full runtime: all pallet configs, construct_runtime!, runtime APIs |
| `roc-chain/node/src/chain_spec.rs` | Genesis configs: dev, local, blocroc (4 validators) |
| `roc-chain/node/src/command.rs` | CLI command routing (loads chain specs by ID) |
| `roc-chain/node/src/service.rs` | Node service: Aura block production + GRANDPA finality |
| `Cargo.toml` | Workspace root: all polkadot-sdk deps via git tag |
| `scripts/start-network.sh` | Launch 4-validator testnet |
| `scripts/stop-network.sh` | Kill all testnet nodes |
| `CLAUDE.md` | Coding conventions and project rules for AI assistants |
