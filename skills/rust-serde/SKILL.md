---
name: rust-serde
description: >
  Serde derive macros, custom serializers/deserializers,
  enum representations (tagged, untagged, adjacently tagged),
  serde attributes (rename, default, flatten, skip, with),
  zero-copy deserialization, format selection (JSON, TOML,
  bincode), DTO patterns, and feature-gating serde in
  libraries. Use when working with serialization,
  deserialization, JSON APIs, config files, wire formats,
  or data conversion between Rust types and external
  representations.
---

# Serde & Serialization

## Core Question

**How does this data cross the boundary between Rust
types and external formats?**

Every `Serialize` / `Deserialize` derive is a contract
with the outside world. The shape of your Rust types
becomes a wire format, a config schema, or a storage
layout. Get the representation wrong and you break
every consumer.

---

## Error -> Design Question

| Symptom | Don't Just Say | Ask Instead |
| --- | --- | --- |
| "expected struct, found string" | "Fix the JSON" | Is your enum representation correct for this format? |
| "missing field `x`" | "Add the field" | Should this field have a default or be Optional? |
| "duplicate field `x`" | "Remove one" | Is `#[serde(flatten)]` creating key collisions? |
| "unknown field `x`" | "Ignore it" | Should you `deny_unknown_fields` to catch drift? |
| Custom deserializer panics | "Fix the input" | Is parse-don't-validate the right approach here? |
| "data did not match any variant" | "Add a variant" | Do you need `#[serde(other)]` for forward compat? |

---

## Quick Decisions

| Situation | Reach For | Why |
| --- | --- | --- |
| New struct for JSON API | `#[derive(Serialize, Deserialize)]` | Derive covers 90% of cases |
| Enum over the wire | `#[serde(tag = "type")]` | Internally tagged is cleanest for JSON APIs |
| Rust snake_case to JSON camelCase | `#[serde(rename_all = "camelCase")]` | One attribute on the container |
| Optional fields in input | `Option<T>` + `#[serde(default)]` | Missing keys become `None`, not errors |
| Newtype wrapper (e.g., `UserId(Uuid)`) | `#[serde(transparent)]` | Serializes as the inner type |
| Reject unexpected keys | `#[serde(deny_unknown_fields)]` | Catches typos and schema drift early |
| Feature-gate serde in a library | `serde` feature + optional dep | Consumers opt in, no forced dependency |
| Flatten nested config | `#[serde(flatten)]` | Merges child fields into parent — watch for key collisions and perf cost |
| Zero-copy deserialization | `&'de str` or `Cow<'de, str>` | Borrows from input buffer, avoids allocation |
| Date/time custom format | `#[serde(with = "module")]` | Isolates format logic in a helper module |
| Sensitive fields (passwords, keys) | `#[serde(skip)]` | Never serialize secrets to the wire |
| Default values for missing fields | `#[serde(default = "path::to_fn")]` | Provides fallback without Option wrapping |
| Validate on deserialize | Custom `Deserialize` impl or wrapper | Enforces invariants at the boundary |
| JSON API server | `serde_json` | Standard, well-tested, good errors |
| Config files | `toml` or `serde_yaml` | Human-friendly formats for configuration |
| High-throughput parsing | `simd-json` with serde compat | SIMD-accelerated, drop-in replacement |

---

## Enum Representation Strategies

Serde supports four enum encodings. The default
(externally tagged) is rarely what you want for JSON
APIs.

```rust
#[derive(Serialize, Deserialize)]
enum Message {
    Text { body: String },
    Image { url: String, width: u32 },
}
```

| Strategy | Attribute | JSON Output |
| --- | --- | --- |
| Externally tagged | *(default)* | `{"Text": {"body": "hi"}}` |
| Internally tagged | `#[serde(tag = "type")]` | `{"type": "Text", "body": "hi"}` |
| Adjacently tagged | `#[serde(tag = "t", content = "c")]` | `{"t": "Text", "c": {"body": "hi"}}` |
| Untagged | `#[serde(untagged)]` | `{"body": "hi"}` |

**Use internally tagged** for JSON APIs — it matches
how most languages model discriminated unions.

**Use adjacently tagged** when you need versioned
payloads or the content can be any type.

**Use untagged** only as a last resort — error messages
are poor and it tries variants in order.

See `references/enums.md` for decision matrix and
forward-compatibility patterns.

---

## The Attribute Quick Reference

| Attribute | Level | Effect |
| --- | --- | --- |
| `rename_all` | Container | Renames all fields/variants |
| `deny_unknown_fields` | Container | Rejects unexpected keys |
| `default` | Container/Field | Uses Default::default for missing fields |
| `tag` / `content` | Container | Controls enum representation |
| `transparent` | Container | Delegates to single inner field |
| `rename` | Field/Variant | Renames one field or variant |
| `skip` | Field/Variant | Excludes from both ser and de |
| `flatten` | Field | Merges nested struct fields inline |
| `with` | Field | Custom ser/de via module path |
| `alias` | Field/Variant | Accepts alternative name on input |
| `default = "path"` | Field | Custom default value function |
| `other` | Variant | Catch-all for unknown variants |

See `references/attributes.md` for full details and
code examples.

---

## Feature-Gating Serde in Libraries

Library crates should not force `serde` on consumers.
Use an optional feature:

```toml
[features]
default = []
serde = ["dep:serde"]

[dependencies]
serde = { version = "1", features = ["derive"], optional = true }
```

```rust
#[cfg_attr(feature = "serde", derive(
    serde::Serialize, serde::Deserialize,
))]
pub struct Config {
    pub name: String,
    pub timeout_ms: u64,
}
```

This pattern keeps the dependency tree lean for
consumers who don't need serialization.

---

## Usage Scenarios

**Scenario 1:** "I'm building JSON API request/response
types"
-> Derive `Serialize` + `Deserialize`, use
`rename_all = "camelCase"`, internally tagged enums,
and `deny_unknown_fields` on input types. Keep response
types permissive, input types strict.
See `references/attributes.md`.

**Scenario 2:** "I need to parse a TOML config file into
typed structs"
-> Derive `Deserialize`, use `#[serde(default)]` for
optional settings, nested structs for sections. Validate
invariants after deserialization or in a custom impl.
See `references/custom.md`.

**Scenario 3:** "My domain types differ from my wire
format"
-> Create separate DTO structs with serde derives.
Convert between domain and DTO using `From`/`Into`.
Never put serde derives on domain types directly.
See `references/custom.md` for DTO patterns.

**Scenario 4:** "I need to deserialize large JSON
payloads fast"
-> Use `simd-json` with serde compat, `&'de str` for
zero-copy strings, and avoid `#[serde(flatten)]` which
disables struct-level optimizations.
See `references/custom.md` for zero-copy patterns.

---

## Reference Files

| File | Read When |
| --- | --- |
| [references/attributes.md](references/attributes.md) | Container, field, and variant attributes with examples |
| [references/enums.md](references/enums.md) | Enum representation strategies, decision matrix, forward compatibility |
| [references/custom.md](references/custom.md) | Custom ser/de, zero-copy, visitors, serde_with, DTO mapping |

---

## Cross-References

| When | Check |
| --- | --- |
| Newtype wrappers, PhantomData in types | rust-types -> Quick Decisions |
| Feature gating, public API naming | rust-api -> Quick Decisions |
| DTO to domain model mapping | rust-ddd -> Quick Decisions |
| Request/response type placement | rust-architecture -> Quick Decisions |
