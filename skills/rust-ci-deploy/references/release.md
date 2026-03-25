# Release Automation

## cargo-dist Setup

`cargo-dist` generates CI workflows that build
platform-specific binaries and publish GitHub releases.

### Install and initialize

```bash
cargo install cargo-dist
cargo dist init
```

This adds configuration to `Cargo.toml`:

```toml
[workspace.metadata.dist]
cargo-dist-version = "0.22.1"
ci = "github"
installers = ["shell", "powershell", "homebrew"]
targets = [
    "aarch64-apple-darwin",
    "x86_64-apple-darwin",
    "x86_64-unknown-linux-gnu",
    "x86_64-unknown-linux-musl",
    "x86_64-pc-windows-msvc",
]
```

### Generate CI workflow

```bash
cargo dist generate
```

This creates `.github/workflows/release.yml` that
triggers on version tags (`v*`). It builds binaries
for all configured targets and uploads them as
release assets.

### Create a release

```bash
git tag v0.1.0
git push origin v0.1.0
```

cargo-dist handles the rest: build, package, upload.

---

## GitHub Actions Release Workflow (Manual)

If you prefer not to use cargo-dist:

```yaml
name: Release

on:
  push:
    tags: ["v*"]

permissions:
  contents: write

jobs:
  build:
    name: Build (${{ matrix.target }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
          - target: aarch64-apple-darwin
            os: macos-latest
          - target: x86_64-apple-darwin
            os: macos-latest
          - target: x86_64-pc-windows-msvc
            os: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}
      - uses: Swatinem/rust-cache@v2
        with:
          key: release-${{ matrix.target }}

      - name: Install musl tools
        if: contains(matrix.target, 'musl')
        run: sudo apt-get install -y musl-tools

      - name: Build
        run: >
          cargo build --release
          --target ${{ matrix.target }}

      - name: Strip binary (Linux/macOS)
        if: runner.os != 'Windows'
        run: strip target/${{ matrix.target }}/release/myapp

      - name: Package (Unix)
        if: runner.os != 'Windows'
        run: >
          tar -czf myapp-${{ matrix.target }}.tar.gz
          -C target/${{ matrix.target }}/release myapp

      - name: Package (Windows)
        if: runner.os == 'Windows'
        run: >
          Compress-Archive
          -Path target/${{ matrix.target }}/release/myapp.exe
          -DestinationPath myapp-${{ matrix.target }}.zip

      - uses: actions/upload-artifact@v4
        with:
          name: myapp-${{ matrix.target }}
          path: myapp-${{ matrix.target }}.*

  release:
    name: Publish Release
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          merge-multiple: true
      - uses: softprops/action-gh-release@v2
        with:
          files: myapp-*
          generate_release_notes: true
```

---

## Changelog Generation with git-cliff

### Install and configure

```bash
cargo install git-cliff
```

Create `cliff.toml`:

```toml
[changelog]
header = "# Changelog\n\n"
body = """
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits %}
- {{ commit.message | upper_first }} \
  ({{ commit.id | truncate(length=7, end="") }})\
{% endfor %}
{% endfor %}\n
"""
trim = true

[git]
conventional_commits = true
filter_unconventional = true
commit_parsers = [
    { message = "^feat", group = "Features" },
    { message = "^fix", group = "Bug Fixes" },
    { message = "^perf", group = "Performance" },
    { message = "^refactor", group = "Refactoring" },
    { message = "^doc", group = "Documentation" },
    { message = "^test", group = "Testing" },
    { message = "^ci", group = "CI/CD" },
    { message = "^chore", skip = true },
]
```

### Generate changelog

```bash
git-cliff --output CHANGELOG.md
git-cliff --latest --strip header
git-cliff --unreleased
```

### In CI (prepend to release notes)

```yaml
- name: Generate release notes
  run: |
    cargo install git-cliff
    git-cliff --latest --strip header > RELEASE_NOTES.md
- uses: softprops/action-gh-release@v2
  with:
    body_path: RELEASE_NOTES.md
```

---

## cargo publish Workflow

### Dry-run in CI (every PR)

```yaml
  publish-check:
    name: Publish Dry Run
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo publish --dry-run
```

### Publish on release

```yaml
  publish:
    name: Publish to crates.io
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo publish
        env:
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

For workspaces with multiple crates, publish in
dependency order:

```bash
cargo publish -p my-core
cargo publish -p my-api
```

---

## Versioning Strategy

### Libraries: SemVer (required by crates.io)

- **MAJOR:** breaking API changes
- **MINOR:** new features, backward-compatible
- **PATCH:** bug fixes, no API changes

### Applications: SemVer or CalVer

SemVer works for apps with a public API or plugin
system. CalVer (`2025.03.1`) works for apps where
the release date is more meaningful than API
stability.

### Version bumping

```bash
cargo install cargo-release
cargo release patch --execute
cargo release minor --execute
```

`cargo-release` bumps `Cargo.toml`, commits, tags,
and optionally publishes in one command.

---

## Binary Size Optimization

### Cargo.toml profile

```toml
[profile.release]
opt-level = "z"     # Optimize for size
lto = true          # Link-time optimization
codegen-units = 1   # Single codegen unit
panic = "abort"     # No unwinding overhead
strip = true        # Strip debug symbols
```

### Size comparison

| Setting | Typical Effect |
| --- | --- |
| Default release | Baseline |
| `strip = true` | -25% to -50% |
| `lto = true` | -10% to -20% |
| `opt-level = "z"` | -5% to -15% |
| `panic = "abort"` | -5% to -10% |
| `codegen-units = 1` | -0% to -5% |
| All combined | -40% to -70% |

Use `opt-level = 3` instead of `"z"` when runtime
performance matters more than binary size.

---

## Homebrew Formula Generation

cargo-dist can generate Homebrew tap formulas:

```toml
[workspace.metadata.dist]
installers = ["homebrew"]

[workspace.metadata.dist.homebrew]
tap = "myorg/homebrew-tap"
```

On release, cargo-dist creates a PR to your tap
repository with the updated formula.

---

## Cross-Platform Release Matrix

Common target triples for CLI tools:

| Target | OS | Arch | Notes |
| --- | --- | --- | --- |
| `x86_64-unknown-linux-gnu` | Linux | x64 | Most common Linux |
| `x86_64-unknown-linux-musl` | Linux | x64 | Static, portable |
| `aarch64-unknown-linux-gnu` | Linux | ARM64 | AWS Graviton, RPi |
| `x86_64-apple-darwin` | macOS | x64 | Intel Macs |
| `aarch64-apple-darwin` | macOS | ARM64 | Apple Silicon |
| `x86_64-pc-windows-msvc` | Windows | x64 | Standard Windows |

Minimum recommended set for broad coverage:
- `x86_64-unknown-linux-gnu`
- `aarch64-apple-darwin`
- `x86_64-pc-windows-msvc`
