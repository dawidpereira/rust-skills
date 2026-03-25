# Rust Skills for Claude Code

A set of Claude Code skills that help Claude write
better Rust. Each skill covers a core Rust topic —
ownership, error handling, types, async, unsafe, API
design, performance, and code quality — and provides
structured reasoning prompts, quick-decision tables,
and reference material so Claude can make informed
choices instead of guessing.

## Installation

### Claude Code

Add to your project's `.claude/settings.json`:

```json
{
  "skills": {
    "rust-skills": {
      "path": "/path/to/rust-skills/skills"
    }
  }
}
```

Or install each skill individually by copying the skill
directories into your Claude Code skills path.

### Project-Level Usage

Copy `CLAUDE.md` to your Rust project root. It provides
reasoning prompts and routes to the right skill
automatically.

## Skills

| Skill | Focus |
|-------|-------|
| `rust-ownership` | Borrowing, lifetimes, smart pointers, interior mutability |
| `rust-errors` | Result/Option, thiserror/anyhow, error propagation |
| `rust-types` | Generics, traits, newtypes, typestate, dispatch |
| `rust-async` | Tokio, channels, Send/Sync, structured concurrency |
| `rust-unsafe` | Unsafe blocks, FFI, safety documentation |
| `rust-api` | API design, builders, naming, documentation standards |
| `rust-perf` | Memory, compiler optimization, profiling |
| `rust-quality` | Testing, linting, project structure, Cargo.toml |
| `rust-tests` | Unit test strategy, integration tests, test builders |
| `rust-architecture` | Project structure, vertical slices, TUI components |
| `rust-ddd` | Domain-driven design, aggregates, value objects |
| `rust-tracing` | Structured logging, spans, tracing setup, observability |

## Architecture

Each skill uses progressive disclosure:

1. **Description** (~80 words) — always in context, triggers the skill
2. **SKILL.md** (<500 lines) — loaded when triggered:
   reasoning scaffold + quick reference
3. **references/** — loaded on demand: synthesized guides with examples

## Sources

Built on:

- [actionbook/rust-skills](https://github.com/actionbook/rust-skills)
- [leonardomso/rust-skills](https://github.com/leonardomso/rust-skills)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [The Rust Performance Book](https://nnethercote.github.io/perf-book/)

## License

MIT
