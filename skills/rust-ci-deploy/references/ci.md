# CI Configuration

## GitHub Actions — Complete Workflow

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-Dwarnings"

jobs:
  fmt:
    name: Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
        with:
          components: rustfmt
      - run: cargo fmt --all --check

  clippy:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --all-targets --all-features

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: [fmt, clippy]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --all-features

  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rustsec/audit-check@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

  deny:
    name: Dependency Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: EmbarkStudios/cargo-deny-action@v2
```

---

## Matrix Testing (Multiple Rust Versions, OSes)

```yaml
  test-matrix:
    name: Test (${{ matrix.os }}, ${{ matrix.rust }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        rust: [stable, beta]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}
      - uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.os }}-${{ matrix.rust }}
      - run: cargo test --all-features
```

---

## GitLab CI Equivalent

```yaml
image: rust:latest

variables:
  CARGO_HOME: "${CI_PROJECT_DIR}/.cargo"
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-Dwarnings"

cache:
  key:
    files:
      - Cargo.lock
  paths:
    - .cargo/registry/index
    - .cargo/registry/cache
    - target/

stages:
  - check
  - test
  - build

format:
  stage: check
  before_script:
    - rustup component add rustfmt
  script:
    - cargo fmt --all --check

lint:
  stage: check
  before_script:
    - rustup component add clippy
  script:
    - cargo clippy --all-targets --all-features

test:
  stage: test
  script:
    - cargo test --all-features

build:
  stage: build
  script:
    - cargo build --release
  artifacts:
    paths:
      - target/release/myapp
    expire_in: 1 week
```

---

## Caching Details

### Swatinem/rust-cache (GitHub Actions)

Automatically caches `~/.cargo/registry`, `target/`,
and handles key generation from `Cargo.lock`:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    shared-key: "ci"
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

Setting `save-if` to only save on main prevents
PR branches from polluting the cache with partial
dependency sets.

### Cache key strategy

Use `Cargo.lock` hash as the primary key. Fall back
to a prefix key so partial caches are reused:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    key: ${{ matrix.os }}
    cache-on-failure: true
```

---

## cargo hack — Feature Testing

Test that every combination of features compiles:

```bash
cargo install cargo-hack
cargo hack check --feature-powerset --no-dev-deps
```

In CI:

```yaml
  feature-check:
    name: Feature Powerset
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - uses: taiki-e/install-action@cargo-hack
      - run: >
          cargo hack check
          --feature-powerset
          --no-dev-deps
```

For large feature sets, limit depth:

```bash
cargo hack check --feature-powerset --depth 2
```

---

## cargo deny — Configuration

Create `deny.toml` in the project root:

```toml
[advisories]
db-path = "~/.cargo/advisory-db"
db-urls = ["https://github.com/rustsec/advisory-db"]
vulnerability = "deny"
unmaintained = "warn"
yanked = "warn"
notice = "warn"

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Unicode-3.0",
    "Unicode-DFS-2016",
]
copyleft = "deny"

[bans]
multiple-versions = "warn"
wildcards = "deny"
highlight = "all"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
allow-git = []
```

Run all checks:

```bash
cargo deny check advisories licenses bans sources
```

---

## cargo audit — Advisory Handling

Install and run:

```bash
cargo install cargo-audit
cargo audit
```

Ignore a specific advisory (with justification):

```bash
cargo audit --ignore RUSTSEC-2024-XXXX
```

Or in `.cargo/audit.toml`:

```toml
[advisories]
ignore = [
    "RUSTSEC-2024-XXXX",  # No fix available, mitigated by X
]
```

---

## MSRV Testing

Verify the minimum supported Rust version declared
in `Cargo.toml` actually works:

```toml
# Cargo.toml
[package]
rust-version = "1.75"
```

```yaml
  msrv:
    name: MSRV Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: taiki-e/install-action@cargo-hack
      - run: cargo hack check --rust-version
```

`cargo hack check --rust-version` reads the
`rust-version` field and tests against it.

---

## Clippy with Deny Warnings

Set `RUSTFLAGS` to turn warnings into errors:

```yaml
env:
  RUSTFLAGS: "-Dwarnings"
```

Or pass the flag directly:

```bash
cargo clippy --all-targets --all-features -- -D warnings
```

For workspace projects, check all members:

```bash
cargo clippy --workspace --all-targets --all-features
```
