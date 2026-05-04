# Config & Dynamic CLI Routing

Production orchestrators rely heavily on declarative TOML manifests to avoid hardcoding prices/endpoints, and extensive CLI tooling for operational management.

## Dependencies

```toml
[dependencies]
toml = "1.1.2+spec-1.1.0"
serde = { version = "1.0.228", features = ["derive"] }
clap = { version = "4.6.1", features = ["derive"] }
```

## Pattern 1: Nested TOML Parsing (The Catalog)

Parse complex TOML structures directly into Axum application state. This powers dynamic pricing arrays and catalog capabilities.

```rust
use serde::Deserialize;
use std::collections::HashMap;

// The struct representation of reeve.toml
#[derive(Deserialize, Debug, Clone)]
pub struct ReeveConfig {
    pub assets: HashMap<String, AssetConfig>,
    pub tasks: HashMap<String, TaskConfig>,
}

#[derive(Deserialize, Debug, Clone)]
pub struct AssetConfig {
    pub network: String,
    pub contract: String,
    pub decimals: u8,
}

#[derive(Deserialize, Debug, Clone)]
pub struct TaskConfig {
    pub title: String,
    pub input_schema: String,
    // The [[tasks.X.accepts]] array maps to Vec<AcceptsConfig>
    #[serde(default)]
    pub accepts: Vec<AcceptsConfig>,
}

#[derive(Deserialize, Debug, Clone)]
pub struct AcceptsConfig {
    pub asset: String,
    pub amount: Option<String>,
    pub amount_usd: Option<String>,
}

// Loading the configuration:
pub fn load_config() -> ReeveConfig {
    let toml_str = std::fs::read_to_string("reeve.toml").expect("Failed to read reeve.toml");
    toml::from_str(&toml_str).expect("Failed to parse TOML")
}
```

## Pattern 2: Clap CLI Subcommands

Use `clap` for operational binaries. The Microservices/Monolith pattern reduces to simply switching which `Command` is invoked at startup.

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "reeve-cli", version, about = "Ops and migrations CLI")]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    /// Check configuration validity
    Config {
        #[arg(long, default_value = "reeve.toml")]
        file: String,
    },
    /// Run database or infrastructure migrations
    Migrate {
        resource: String, // e.g., "db", "mq", "s3"
        #[arg(long)]
        from: String,
        #[arg(long)]
        to: String,
        #[arg(long)]
        batch: Option<usize>,
        #[arg(long, action)]
        dry_run: bool,
    },
    /// Start the API server
    Serve {
        #[arg(long, default_value = "0.0.0.0:8080")]
        bind: String,
    },
    /// Start the background worker
    Work {
        #[arg(long)]
        queue: String,
    }
}

// In main.rs:
// let cli = Cli::parse();
// match cli.command {
//     Commands::Migrate { resource, from, to, batch, dry_run } => { ... }
//     Commands::Serve { bind } => { ... }
//     _ => { ... }
// }
```
