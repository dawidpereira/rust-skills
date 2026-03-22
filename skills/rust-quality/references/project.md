# Project Structure Reference

## lib.rs / main.rs Split

Keep `main.rs` as a thin entry point. All application logic belongs in `lib.rs` so
integration tests can access it.

```rust
// src/main.rs
use my_app::{run, Config};

fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    run(config)
}

// src/lib.rs
pub mod config;
pub mod database;
pub mod handlers;

pub use config::Config;

pub fn run(config: Config) -> anyhow::Result<()> {
    let db = database::connect(&config.db_url)?;
    let app = handlers::build_app(db);
    app.run()
}
```

Integration tests can now test `run()` and all public modules directly.

## Feature-Based Module Organization

Group related code by feature, not by type.

```
# Bad: type-based (scattered)
src/
├── controllers/
│   ├── user_controller.rs
│   └── order_controller.rs
├── models/
│   ├── user.rs
│   └── order.rs
└── services/
    ├── user_service.rs
    └── order_service.rs

# Good: feature-based (cohesive)
src/
├── user/
│   ├── mod.rs
│   ├── model.rs
│   ├── repository.rs
│   ├── service.rs
│   └── handler.rs
├── order/
│   ├── mod.rs
│   ├── model.rs
│   └── service.rs
├── shared/
│   ├── mod.rs
│   ├── error.rs
│   └── database.rs
└── lib.rs
```

Adding a feature means creating one directory. Deleting a feature means removing one
directory. Understanding a feature means reading one directory.

## Flat Structure for Small Projects

Do not over-organize projects with fewer than 10 files.

```
src/
├── main.rs
├── lib.rs
├── config.rs
├── database.rs
├── user.rs
└── error.rs
```

When to add structure:

| File count | Structure |
|------------|-----------|
| < 10 | Flat in `src/` |
| 10-20 | Group by feature |
| 20+ | Feature folders with submodules |

Signs of over-structure: folders with 1-2 files, `mod.rs` that only re-exports,
deep nesting for simple concepts.

## mod.rs vs Adjacent File

Two styles for multi-file modules. Pick one and use it consistently.

```
# mod.rs style (complex modules)
src/user/
├── mod.rs
├── model.rs
└── repository.rs

# Adjacent file style (simple modules)
src/
├── user.rs           # Module root
└── user/
    ├── model.rs
    └── repository.rs
```

Enforce consistency with clippy:

```toml
[lints.clippy]
mod_module_files = "warn"          # Enforces mod.rs style
# OR
self_named_module_files = "warn"   # Enforces adjacent style
```

## Visibility: pub(crate)

Use `pub(crate)` for items shared within the crate but hidden from external users.

```rust
// Bad: everything public
pub struct InternalState {
    pub buffer: Vec<u8>,
}

// Good: crate-internal visibility
pub(crate) struct InternalState {
    pub(crate) buffer: Vec<u8>,
}

pub struct Widget {
    state: InternalState,
}

impl Widget {
    pub fn new() -> Self {
        Self { state: InternalState { buffer: Vec::new() } }
    }
}
```

| Visibility | Accessible from |
|------------|-----------------|
| `pub` | Everywhere |
| `pub(crate)` | Current crate only |
| `pub(super)` | Parent module only |
| `pub(in path)` | Specific module path |
| (private) | Current module only |

## Visibility: pub(super)

Use `pub(super)` for helpers shared between sibling submodules but hidden from the
rest of the crate.

```rust
// src/parser/lexer.rs
pub(super) fn tokenize(input: &str) -> Vec<Token> { /* ... */ }

// src/parser/ast.rs
use super::lexer::tokenize;

pub(super) fn build_ast(input: &str) -> Ast {
    let tokens = tokenize(input);
    // ...
}

// src/parser/mod.rs
mod lexer;
mod ast;

pub fn parse(input: &str) -> Ast {
    ast::build_ast(input)
}
```

## pub use Re-Exports

Create a clean public API by re-exporting from `mod.rs`:

```rust
// src/user/mod.rs
mod model;
mod repository;
mod service;

pub use model::{User, UserId, CreateUserRequest};
pub use service::UserService;
pub(crate) use repository::UserRepository;
```

Users import `my_crate::user::User` without knowing the internal file structure.
Re-exports decouple public API from module layout.

## Prelude Module

For libraries, create a prelude for convenient glob importing. Include only types users
need in almost every file. Be conservative -- removing items is a breaking change.

```rust
pub mod prelude {
    pub use crate::{Client, Config, Error};
    pub use crate::traits::{Handler, Middleware};
}
// Users write: use my_crate::prelude::*;
```

## Multiple Binaries: src/bin/

Place multiple binaries in `src/bin/`. Each file becomes a binary target automatically.

```
src/
├── lib.rs
└── bin/
    ├── server.rs
    └── cli.rs
```

```rust
// src/bin/server.rs
use my_project::{config, database};

fn main() {
    let config = config::load();
    let db = database::connect(&config);
}
```

For complex binaries with multiple files, use directories with `main.rs`
(e.g., `src/bin/server/main.rs`).

## Workspaces

Use workspaces for large projects with multiple crates:

```toml
# Root Cargo.toml
[workspace]
members = ["crates/*"]
resolver = "2"
```

### Dependency Inheritance

Define shared dependencies at the workspace root:

```toml
# Root Cargo.toml
[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
anyhow = "1.0"

# Member Cargo.toml
[dependencies]
serde = { workspace = true }
tokio = { workspace = true }
```

### Lint Inheritance

Define lints once:

```toml
# Root Cargo.toml
[workspace.lints.clippy]
correctness = "deny"
suspicious = "warn"
style = "warn"
complexity = "warn"
perf = "warn"

# Member Cargo.toml
[lints]
workspace = true
```

Split into a workspace when:
- Build times are slow and you want incremental compilation.
- Different parts have different dependency requirements.
- You want to publish crates independently.
- Team members work on separate crate boundaries.
