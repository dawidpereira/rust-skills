# Custom Serialization

## When to Write Custom Ser/De

Derive works for most types. Write custom
implementations when:

- You need **validation on deserialize** (e.g., a
  non-empty string, a positive integer, an email).
- The external format is **legacy or non-standard**
  (e.g., timestamps as integers, booleans as "yes"/"no").
- You want **zero-copy deserialization** borrowing from
  the input buffer.
- A third-party type doesn't implement serde and
  `#[serde(remote)]` doesn't fit.

---

## Custom Deserialize for Validated Newtypes

Parse-don't-validate: reject invalid data at the
deserialization boundary.

```rust
use serde::de::{self, Deserialize, Deserializer};

pub struct NonEmptyString(String);

impl NonEmptyString {
    pub fn new(s: String) -> Result<Self, &'static str> {
        if s.is_empty() {
            Err("string must not be empty")
        } else {
            Ok(Self(s))
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl<'de> Deserialize<'de> for NonEmptyString {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        NonEmptyString::new(s).map_err(de::Error::custom)
    }
}
```

The same pattern applies to `Port(u16)` that rejects
zero, `EmailAddress` that validates format, etc. Once
deserialized, the type is guaranteed valid — no runtime
checks needed downstream.

---

## serialize_with / deserialize_with Helpers

For one-off format overrides without a full impl,
write helper functions in a module:

```rust
mod unix_timestamp {
    use chrono::{DateTime, Utc};
    use serde::{self, Deserialize, Deserializer, Serializer};

    pub fn serialize<S>(
        dt: &DateTime<Utc>,
        ser: S,
    ) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        ser.serialize_i64(dt.timestamp())
    }

    pub fn deserialize<'de, D>(
        de: D,
    ) -> Result<DateTime<Utc>, D::Error>
    where
        D: Deserializer<'de>,
    {
        let ts = i64::deserialize(de)?;
        DateTime::from_timestamp(ts, 0)
            .ok_or_else(|| {
                serde::de::Error::custom("invalid timestamp")
            })
    }
}

#[derive(Serialize, Deserialize)]
struct AuditEntry {
    action: String,
    #[serde(with = "unix_timestamp")]
    occurred_at: DateTime<Utc>,
}
```

---

## The Visitor Pattern

For complex deserialization logic, implement a
`Visitor`. The deserializer calls the visitor method
matching the input format (string, map, seq, etc.):

```rust
use serde::de::{self, Deserializer, Visitor};
use std::fmt;

struct PortVisitor;

impl<'de> Visitor<'de> for PortVisitor {
    type Value = u16;

    fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.write_str("a port number 1-65535 or a string")
    }

    fn visit_u64<E: de::Error>(
        self,
        v: u64,
    ) -> Result<Self::Value, E> {
        u16::try_from(v)
            .map_err(|_| E::custom("port out of range"))
            .and_then(|p| {
                if p == 0 {
                    Err(E::custom("port must be non-zero"))
                } else {
                    Ok(p)
                }
            })
    }

    fn visit_str<E: de::Error>(
        self,
        v: &str,
    ) -> Result<Self::Value, E> {
        v.parse::<u16>()
            .map_err(|_| E::custom("invalid port string"))
    }
}
```

Use a visitor when you need to accept **multiple input
types** (string or integer) or when validation logic
requires branching on the input kind.

---

## Zero-Copy Deserialization

When parsing large payloads, avoid allocating strings
by borrowing from the input buffer:

```rust
#[derive(Deserialize)]
struct LogEntry<'a> {
    #[serde(borrow)]
    message: &'a str,

    #[serde(borrow)]
    source: Cow<'a, str>,

    level: u8,
}
```

- `&'de str` borrows directly — zero allocation, but
  the input buffer must outlive the struct.
- `Cow<'de, str>` borrows when possible, allocates
  when the deserializer must transform the data
  (e.g., unescape a JSON string with `\n`).
- Only works with formats that support borrowing
  (`serde_json::from_str`, not `from_reader`).

**Use `Cow` by default** — it borrows when it can and
falls back to owned when it must. Pure `&str` is an
optimization for when you control the input format.

---

## The serde_with Crate

`serde_with` provides ready-made transformations for
common patterns, avoiding manual `serialize_with`
boilerplate:

```rust
use serde_with::{serde_as, DisplayFromStr};

#[serde_as]
#[derive(Serialize, Deserialize)]
struct Config {
    #[serde_as(as = "DisplayFromStr")]
    port: u16,  // ser/de as string via Display/FromStr
}
```

Common adapters:

| Adapter | Effect |
| --- | --- |
| `DisplayFromStr` | Ser/de via `Display`/`FromStr` |
| `DurationSeconds` | Duration as integer seconds |
| `TimestampSeconds` | DateTime as unix timestamp |
| `NoneAsEmptyString` | `None` <-> `""` |
| `BoolFromInt` | `true` <-> `1`, `false` <-> `0` |
| `VecSkipError` | Skip elements that fail to parse |
| `Map<K, V>` | Vec of tuples as a JSON object |

Prefer `serde_with` over hand-written modules for
standard transformations — it's well-tested and
self-documenting.

---

## DTO <-> Domain Model Patterns

Serde derives belong on DTOs (data transfer objects),
not on domain types. Domain types enforce invariants;
DTOs match the wire format.

```rust
// Domain type — no serde, enforces invariants
pub struct Order {
    id: OrderId,
    items: Vec<LineItem>,  // guaranteed non-empty
    total: Money,          // guaranteed positive
}

// DTO — matches the JSON shape
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct OrderDto {
    pub id: String,
    pub items: Vec<LineItemDto>,
    pub total_cents: i64,
}

// Conversion with validation
impl TryFrom<OrderDto> for Order {
    type Error = DomainError;

    fn try_from(dto: OrderDto) -> Result<Self, Self::Error> {
        let id = OrderId::parse(dto.id)?;
        let items: Vec<LineItem> = dto.items
            .into_iter()
            .map(LineItem::try_from)
            .collect::<Result<_, _>>()?;

        if items.is_empty() {
            return Err(DomainError::EmptyOrder);
        }

        let total = Money::from_cents(dto.total_cents)?;
        Ok(Order { id, items, total })
    }
}

// Infallible conversion for responses
impl From<&Order> for OrderDto {
    fn from(order: &Order) -> Self {
        OrderDto {
            id: order.id.to_string(),
            items: order.items.iter().map(Into::into).collect(),
            total_cents: order.total.as_cents(),
        }
    }
}
```

**Why separate DTOs?**
- Domain types stay free of serialization concerns.
- Wire format can evolve independently (versioning).
- Validation happens at the boundary, not deep in
  business logic.
- Different APIs can use different DTOs for the same
  domain type.

---

## Format Selection Guide

| Format | Crate | Use When |
| --- | --- | --- |
| JSON | `serde_json` | HTTP APIs, browser interop |
| TOML | `toml` | Config files (Cargo.toml style) |
| YAML | `serde_yaml` | Config files (k8s style) |
| bincode | `bincode` | Rust-to-Rust IPC, caching |
| postcard | `postcard` | Embedded / no_std, compact |
| MessagePack | `rmp-serde` | Compact binary, cross-language |
| CBOR | `ciborium` | IoT, constrained environments |
| simd-json | `simd-json` | High-throughput JSON parsing |

For JSON APIs, `serde_json` is the standard. Consider
`simd-json` only when profiling shows JSON parsing is
a bottleneck — it requires `unsafe` internally and
mutable input buffers.
