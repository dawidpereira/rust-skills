---
name: rust-ci-deploy
description: >
  CI pipelines, GitHub Actions, caching, cross-compilation,
  Docker builds, release automation, cargo-dist, cargo-audit,
  cargo-deny, static linking, musl. Trigger for CI/CD,
  deployment, and release tasks.
---

# CI, Docker & Release

## Core Question

**How does this code get from a passing commit to a
production artifact that users can trust?**

Every merge should produce a verified, reproducible
artifact. CI proves correctness, Docker packages the
artifact, and release automation delivers it. If any
stage is manual, it will eventually be skipped.

---

## Error → Design Question

| Symptom | Don't Just Say | Ask Instead |
| --- | --- | --- |
| CI takes 15+ minutes | "Use a faster runner" | Are you caching the target directory and registry? |
| "linking failed" in CI | "Install the library" | Is the target platform's linker/libs installed? |
| Docker image is 1GB+ | "Use a smaller base" | Are you using multi-stage builds with scratch/distroless? |
| Binary doesn't run on target | "Recompile it" | Are you compiling for the right target triple? |
| Vulnerability in dependency | "Ignore the advisory" | Is cargo-audit in CI, and is the advisory ignorable? |
| Release artifacts missing | "Run the workflow again" | Is the release workflow triggered on tags? |

---

## Quick Decisions

| Situation | Reach For | Why |
| --- | --- | --- |
| Rust CI pipeline | fmt check + clippy + test + audit | Ordered by speed: fast failures first |
| Caching in GitHub Actions | `Swatinem/rust-cache` action | Caches target/ and registry automatically |
| Caching in GitLab CI | Cache `target/` and `~/.cargo/registry` | Avoids recompilation on every push |
| Testing all feature combos | `cargo hack check --feature-powerset` | Catches feature-gated compile errors |
| Cross-compiling CLI tool | `cross` (Docker-based, multi-target) | No manual toolchain setup per target |
| Static binary for Linux | `x86_64-unknown-linux-musl` target | No glibc dependency, runs anywhere |
| Minimal Docker image | Multi-stage: `rust:slim` + scratch/distroless | Builder compiles, runtime has only the binary |
| Automated releases | `cargo-dist` for GitHub releases | Builds binaries, generates installers |
| Changelog generation | `git-cliff` with conventional commits | Deterministic changelogs from git history |
| Crate publishing | `cargo publish` with `--dry-run` in CI | Catches packaging errors before release |
| Dependency audit | `cargo audit` in CI, fail on vulns | Blocks merges with known vulnerabilities |
| License compliance | `cargo deny check licenses` | Prevents accidental copyleft inclusion |
| Supply chain security | `cargo deny check advisories + bans` | Blocks yanked or banned crates |
| MSRV verification | `cargo hack check --rust-version` | Proves the declared MSRV actually works |
| Binary size optimization | strip, LTO, `opt-level = "z"` | Smaller downloads, faster cold starts |

---

## The CI Pipeline

Order matters. Each stage gates the next, and fast
checks run first to minimize wasted compute:

1. **Format** (`cargo fmt --check`) — instant, catches
   style drift before anything compiles.
2. **Lint** (`cargo clippy -- -D warnings`) — catches
   logic issues and anti-patterns early.
3. **Test** (`cargo test`) — proves correctness.
4. **Audit** (`cargo audit`, `cargo deny`) — checks
   dependencies for vulnerabilities and license issues.
5. **Build** (`cargo build --release`) — produces the
   artifact only after all checks pass.

If format fails, there is no point running tests.
If tests fail, there is no point building a release.

---

## Docker Multi-Stage Build

Separate compilation from runtime. The builder stage
has the full toolchain; the runtime stage has only
the binary:

```dockerfile
FROM rust:1-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release && rm -rf src target/release/.fingerprint
COPY src ./src
RUN cargo build --release

FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/target/release/myapp /
ENTRYPOINT ["/myapp"]
```

The dummy-build trick caches dependencies as a
separate layer. Source changes only rebuild app code.

See `references/docker.md` for musl static linking,
.dockerignore, and health check patterns.

---

## Caching Strategy

**Cache these:**
- `~/.cargo/registry/index`
- `~/.cargo/registry/cache`
- `target/`

**Never cache these:**
- `~/.cargo/registry/src` (re-extracted from cache)
- `target/debug/incremental` (not portable across CI)

**Cache key:** hash of `Cargo.lock` + Rust toolchain
version. A lockfile change means dependency rebuild.

See `references/ci.md` for full cache configuration.

---

## Usage Scenarios

**Scenario 1:** "I'm setting up CI for a new Rust project"
→ Start with fmt + clippy + test in GitHub Actions.
Add `Swatinem/rust-cache` for caching. Add cargo-audit
once dependencies stabilize.
See `references/ci.md` for complete workflow YAML.

**Scenario 2:** "I need to deploy a Rust service in Docker"
→ Use a multi-stage Dockerfile with `rust:slim` as
builder and distroless as runtime. Copy `Cargo.toml`
first for layer caching.
See `references/docker.md` for production patterns.

**Scenario 3:** "I want to release CLI binaries for multiple platforms"
→ Use `cargo-dist` to generate GitHub Actions workflows
that build and upload binaries on tag push. Configure
targets in `Cargo.toml`.
See `references/release.md` for cargo-dist setup.

**Scenario 4:** "I need to audit dependencies for vulnerabilities"
→ Add `cargo audit` and `cargo deny` to CI. Configure
`deny.toml` for license and advisory policies.
See `references/ci.md` for deny.toml configuration.

---

## Reference Files

| File | Read When |
| --- | --- |
| [references/ci.md](references/ci.md) | GitHub Actions workflow, GitLab CI, caching, cargo-deny, cargo-audit, MSRV testing |
| [references/docker.md](references/docker.md) | Multi-stage Dockerfile, musl static linking, distroless, .dockerignore, Docker Compose |
| [references/release.md](references/release.md) | cargo-dist, git-cliff, cargo publish, versioning, binary optimization, cross-platform releases |

---

## Cross-References

| When | Check |
| --- | --- |
| Cargo.toml defaults, clippy lint setup | rust-quality → Quick Decisions |
| Release profile optimization, LTO | rust-perf → Quick Decisions |
| Production observability in containers | rust-tracing → Quick Decisions |
| Test organization for CI | rust-tests → Quick Decisions |
