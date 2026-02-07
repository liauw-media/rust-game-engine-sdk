# rust-game-engine-sdk

Rust SDK for building GLI-19 compliant slot game engines.

## Overview

This crate provides the core traits, types, and utilities for implementing game engines that integrate with the Omnitronix RGS (Remote Gaming Server).

**Reference Implementation:** [`@omnitronix/game-engine-contract`](https://github.com/liauw-media/omnitronix-monorepo) (TypeScript)

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     rust-game-engine-sdk                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Traits:                                                    │
│  ├── GameEngine<TPublic, TPrivate, TOutcome>               │
│  ├── RngClient                                              │
│  └── Logger                                                 │
│                                                             │
│  Types:                                                     │
│  ├── GameActionCommand                                      │
│  ├── CommandProcessingResult                                │
│  ├── GameEngineInfo                                         │
│  ├── RngOutcome / RngOutcomeRecord                         │
│  ├── TriggeredBonus / BonusProgress                        │
│  └── CommandType (enum)                                     │
│                                                             │
│  Utilities:                                                 │
│  ├── rng_outcome_mapper (wire format conversion)           │
│  ├── canonicalize_for_hash (GLI-19 audit)                  │
│  └── debug_command_filter                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Core Traits

### `GameEngine` Trait

```rust
/// Core trait all game engines must implement.
/// Game engines are stateless processors: given state + command, return new state.
pub trait GameEngine<TPublic, TPrivate, TOutcome>: Send + Sync {
    /// Process a game command and return the resulting state.
    async fn process_command(
        &self,
        public_state: Option<TPublic>,
        private_state: Option<TPrivate>,
        command: GameActionCommand,
    ) -> Result<CommandProcessingResult<TPublic, TPrivate, TOutcome>, GameEngineError>;

    /// Get metadata about this game engine.
    fn get_game_engine_info(&self) -> GameEngineInfo;
}
```

### `RngClient` Trait

```rust
/// Interface for RNG providers (local for testing, remote for production).
#[async_trait]
pub trait RngClient: Send + Sync {
    /// Generate a single random integer in range [min, max]
    async fn get_single_number(&self, min: i64, max: i64, seed: Option<i64>)
        -> Result<RngSingleResponse, RngError>;

    /// Generate multiple random integers
    async fn get_batch_numbers(&self, min: i64, max: i64, count: usize, seed: Option<i64>)
        -> Result<Vec<i64>, RngError>;

    /// Generate multiple random integers with individual seeds (for audit trail)
    async fn get_batch_numbers_with_seeds(&self, min: i64, max: i64, count: usize, seed: Option<i64>)
        -> Result<Vec<RngSingleResponse>, RngError>;

    /// Generate a single random float in range [0, 1)
    async fn get_single_float(&self, seed: Option<i64>)
        -> Result<RngSingleResponse, RngError>;

    /// Generate multiple random floats
    async fn get_batch_floats(&self, count: usize, seed: Option<i64>)
        -> Result<RngBatchResponse, RngError>;
}
```

## Core Types

### Commands

```rust
/// Incoming command from RGS to game engine
pub struct GameActionCommand {
    pub id: String,
    pub command_type: CommandType,
    pub payload: Option<serde_json::Value>,
}

/// Command types
pub enum CommandType {
    Spin,
    GetSymbols,
    StartBonusRound,
    BonusSpin,
    // Debug commands (blocked in production)
    DebugTriggerBonus,
    DebugForceWin,
    DebugSetRtp,
    DebugUpdateBonusMeterProgress,
}
```

### Results

```rust
/// Response from game engine back to RGS
pub struct CommandProcessingResult<TPublic, TPrivate, TOutcome> {
    pub success: bool,
    pub public_state: TPublic,
    pub private_state: TPrivate,
    pub outcome: Option<TOutcome>,
    pub message: Option<String>,
    pub rng_outcome: Option<RngOutcome>,
}

/// Game engine metadata
pub struct GameEngineInfo {
    pub game_code: String,
    pub version: String,
    pub rtp: f64,
    pub game_type: GameType,
    pub game_name: String,
    pub provider: String,
}
```

### RNG Audit Trail

```rust
/// Single RNG call result (for GLI-19 audit)
pub struct RngOutcomeRecord {
    pub result: i64,
    pub seed: i64,
    pub min: i64,
    pub max: i64,
}

/// Collection of RNG calls from one command
pub type RngOutcome = HashMap<String, RngOutcomeRecord>;

/// Canonicalize RNG outcome for deterministic hashing
pub fn canonicalize_for_hash(outcome: &RngOutcome) -> String {
    // Returns deterministic JSON for GLI-19 compliance
}
```

### Bonus Types

```rust
/// Triggered bonus info
pub struct TriggeredBonus {
    pub bonus_id: String,
    pub bonus_type: BonusType,
    pub bet_amount: Decimal,
}

/// Bonus progress tracking
pub struct BonusProgress {
    pub completed: u32,
    pub remaining: u32,
    pub total: u32,
}

pub enum BonusType {
    FreeSpins,
    WheelBonus,
    PickBonus,
    // Add more as needed
}
```

## Features

- `wasm` - Compile to WebAssembly for browser/edge deployment
- `grpc` - Include gRPC service definitions
- `local-rng` - Include local RNG for testing (ChaCha20-based)

## Usage

```toml
[dependencies]
rust-game-engine-sdk = { git = "https://github.com/liauw-media/rust-game-engine-sdk" }
```

```rust
use rust_game_engine_sdk::prelude::*;

pub struct MySlotEngine {
    rng_client: Box<dyn RngClient>,
}

impl GameEngine<MyPublicState, MyPrivateState, SpinOutcome> for MySlotEngine {
    async fn process_command(
        &self,
        public_state: Option<MyPublicState>,
        private_state: Option<MyPrivateState>,
        command: GameActionCommand,
    ) -> Result<CommandProcessingResult<MyPublicState, MyPrivateState, SpinOutcome>, GameEngineError> {
        match command.command_type {
            CommandType::Spin => self.handle_spin(public_state, private_state).await,
            CommandType::StartBonusRound => self.start_bonus(public_state, private_state).await,
            _ => Err(GameEngineError::UnsupportedCommand),
        }
    }

    fn get_game_engine_info(&self) -> GameEngineInfo {
        GameEngineInfo {
            game_code: "my-slot".into(),
            version: "1.0.0".into(),
            rtp: 96.5,
            game_type: GameType::Slot,
            game_name: "My Slot Game".into(),
            provider: "Omnitronix".into(),
        }
    }
}
```

## Reference Files

| Rust Module | TypeScript Reference |
|-------------|---------------------|
| `src/traits/game_engine.rs` | `game-engine-contract/src/core/game-engine.interface.ts` |
| `src/traits/rng_client.rs` | `happy-panda-game-engine/src/rng/rng-client.interface.ts` |
| `src/types/command.rs` | `game-engine-contract/src/core/command.interface.ts` |
| `src/types/rng_outcome.rs` | `game-engine-contract/src/core/rng-outcome.interface.ts` |
| `src/types/bonus.rs` | `game-engine-contract/src/bonus/bonus.types.ts` |

## GLI-19 Compliance

This SDK is designed for GLI-19 certified gaming:

- All RNG calls are captured with seeds for replay/verification
- State is immutable (public vs private separation)
- Debug commands are filtered in production mode
- Deterministic canonicalization for audit hashes

## License

MIT
