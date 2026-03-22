---
name: rust-errors
description: >
  Rust error handling patterns with Result, Option, thiserror, and anyhow. Use whenever
  implementing error types, adding error propagation with ?, choosing between panic and
  Result, or working with .unwrap()/.expect(). Covers library vs application error strategy,
  error chaining, context, and custom error types. Also use when seeing unwrap abuse or
  unclear error propagation in code review.
---

# Error Handling

## Core Question

**Is this failure expected or a bug?**

This single question drives every error handling decision:

- **Expected** (file not found, invalid input, network timeout)
  → `Result<T, E>`
- **Absence** (optional value, lookup miss) → `Option<T>`
- **Bug** (violated invariant, impossible state) → `panic!` / `expect()`

If you're unsure, it's almost certainly expected — use `Result`.

---

## Error → Design Question

| Symptom                    | Don't Just Say           | Ask Instead                                                  |
| -------------------------- | ------------------------ | ------------------------------------------------------------ |
| Unwrap everywhere          | "Add expect message"     | What should happen when this fails?                          |
| `Box<dyn Error>` return    | "It works"               | Can the caller handle specific errors?                       |
| Error type has 15 variants | "Be thorough"            | Are these all distinct failure modes the caller cares about? |
| `.context()` on every line | "More context is better" | What context does the _caller_ actually need?                |
| Panic in library code      | "Document the panic"     | Should this be a Result instead?                             |

---

## Quick Decisions

| Scenario                                   | Use                                   | Why                                    |
| ------------------------------------------ | ------------------------------------- | -------------------------------------- |
| Library with typed errors callers match on | `thiserror`                           | Generates `Error` impl, typed variants |
| Application-level error handling           | `anyhow`                              | Easy context, no custom types needed   |
| Library public API + internal convenience  | `thiserror` public, `anyhow` internal | Best of both                           |
| Function can fail in expected ways         | `Result<T, E>`                        | Caller decides how to handle           |
| Value may or may not exist                 | `Option<T>`                           | No error info needed                   |
| Invariant violated (bug in code)           | `panic!` / `unreachable!`             | Should never happen in correct code    |
| Test assertions                            | `unwrap()` / `expect()`               | Panics give clear test failures        |
| One-off script / prototype                 | `anyhow::Result` in main              | Quick iteration, good error display    |

---

## The ? Operator

`?` is the backbone of Rust error handling. It propagates
errors up the call stack, converting types via `From`
automatically.

```rust
fn load_config(path: &Path) -> Result<Config, AppError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → AppError via From
    let config: Config = toml::from_str(&content)?; // toml::Error → AppError via From
    Ok(config)
}
```

Add context when the error alone doesn't tell the story:

```rust
use anyhow::Context;

let content = std::fs::read_to_string(path)
    .with_context(|| format!("failed to read config from {}", path.display()))?;
```

---

## Library vs Application

| Context     | Crate                      | Pattern                                     |
| ----------- | -------------------------- | ------------------------------------------- |
| Library     | `thiserror`                | Typed, matchable errors callers can inspect |
| Application | `anyhow`                   | Easy propagation with context chains        |
| Both        | `thiserror` for public API | `anyhow` for internal plumbing              |

### Library Error (thiserror)

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ParseError {
    #[error("invalid syntax at line {line}: {message}")]
    Syntax { line: usize, message: String },

    #[error("unexpected end of file")]
    UnexpectedEof,

    #[error("io error reading input")]
    Io(#[from] std::io::Error),
}
```

### Application Error (anyhow)

```rust
use anyhow::{Context, Result, bail, ensure};

fn run() -> Result<()> {
    let config = load_config("app.toml")
        .context("failed to initialize")?;

    ensure!(config.port > 0, "port must be positive, got {}", config.port);

    if config.debug && config.production {
        bail!("cannot enable debug in production");
    }
    Ok(())
}
```

---

## Error Message Conventions

- **Start lowercase**, no trailing punctuation:
  `"invalid input"` not `"Invalid input."`
- Acronyms and error codes stay uppercase:
  `"DNS lookup failed"`, `"HTTP 503"`
- Context reads naturally when chained:
  `"failed to load config: invalid toml: expected '=' at line 3"`

---

## Usage Scenarios

**Scenario 1:** "I'm writing a library — how should I define errors?"
→ Use `thiserror`. Create an enum with one variant per
failure mode the _caller_ cares about. Use `#[from]` for
automatic conversion from underlying errors. Don't expose
internal error types — wrap them.

**Scenario 2:** "I have unwrap() calls everywhere in my application"
→ Replace with `?` and `anyhow::Result`. Add `.context()`
where the error alone isn't enough to diagnose the problem.
Keep `unwrap()` only in tests and for invariants you've
already validated.

**Scenario 3:** "Should I use expect or return an error here?"
→ If the caller can do something about it (retry, use a
default, report to user), return `Result`. If it means the
program's logic is broken (invariant violation),
`expect("reason this should never fail")` is appropriate.

---

## Reference Files

| File                                               | Read When                                                                                       |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| [error-patterns](references/patterns.md)   | Setting up thiserror/anyhow, error chaining, #[from]/#[source], context patterns, documentation |
| [error-decisions](references/decisions.md) | Deciding panic vs Result, designing custom error types, when-to-use-what scenarios              |

---

## Cross-References

| When                                      | Check                            |
| ----------------------------------------- | -------------------------------- |
| Error types involving ownership/lifetimes | rust-ownership → Quick Decisions |
| Async error handling patterns             | rust-async → Quick Decisions     |
| Documenting # Errors sections             | rust-api → Quick Decisions       |
| Clippy lints for error handling           | rust-quality → Quick Decisions   |
