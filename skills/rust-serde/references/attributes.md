# Serde Attributes Reference

## Container Attributes

Applied to structs or enums with `#[serde(...)]`.

### rename_all

Renames all fields or variants to a given convention:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiResponse {
    user_id: String,      // -> "userId"
    created_at: DateTime,  // -> "createdAt"
}
```

Supported values: `"camelCase"`, `"snake_case"`,
`"PascalCase"`, `"SCREAMING_SNAKE_CASE"`,
`"kebab-case"`, `"SCREAMING-KEBAB-CASE"`.

You can apply different conventions for ser and de:

```rust
#[serde(rename_all(serialize = "camelCase",
                    deserialize = "snake_case"))]
```

---

### deny_unknown_fields

Rejects any JSON key not matching a struct field:

```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct CreateUser {
    name: String,
    email: String,
}
// {"name": "Jo", "email": "a@b", "age": 5} -> error
```

Use on **input types** (requests, configs) to catch
schema drift. Avoid on output types where extra fields
from newer versions should be tolerated.

---

### default

Uses `Default::default()` for every missing field:

```rust
#[derive(Deserialize)]
#[serde(default)]
struct Config {
    port: u16,       // 0 if missing
    debug: bool,     // false if missing
    host: String,    // "" if missing
}
```

Pair with a manual `Default` impl for meaningful
defaults:

```rust
impl Default for Config {
    fn default() -> Self {
        Self {
            port: 8080,
            debug: false,
            host: "localhost".into(),
        }
    }
}
```

---

### tag / content (enum representation)

Controls how enum variants are encoded. See
`enums.md` for full coverage.

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "kind")]
enum Event {
    Click { x: i32, y: i32 },
    Scroll { delta: f64 },
}
// -> {"kind": "Click", "x": 10, "y": 20}
```

---

### transparent

Delegates serialization to the single inner field:

```rust
#[derive(Serialize, Deserialize)]
#[serde(transparent)]
struct UserId(Uuid);
// Serializes as a plain UUID string, not {"0": "..."}
```

Essential for newtype wrappers that should be invisible
on the wire.

---

### bound

Overrides the default trait bounds serde generates:

```rust
#[derive(Serialize)]
#[serde(bound(serialize = "T: Display"))]
struct Wrapper<T> {
    #[serde(serialize_with = "display_ser")]
    value: T,
}
```

Useful when the default `T: Serialize` bound is wrong
because you use a custom serializer.

---

### remote

Derives serde for a type from another crate that
doesn't implement serde:

```rust
mod duration_def {
    use serde::{Serialize, Deserialize};

    #[derive(Serialize, Deserialize)]
    #[serde(remote = "std::time::Duration")]
    pub struct DurationDef {
        secs: u64,
        nanos: u32,
    }
}

#[derive(Serialize, Deserialize)]
struct Config {
    #[serde(with = "duration_def")]
    timeout: std::time::Duration,
}
```

---

## Field Attributes

Applied to individual struct fields.

### rename

```rust
#[derive(Serialize, Deserialize)]
struct User {
    #[serde(rename = "type")]
    kind: String,
}
// -> {"type": "admin"}
```

Handles Rust reserved words and API naming mismatches.

---

### default (field-level)

Provides a fallback when the field is missing:

```rust
#[derive(Deserialize)]
struct Settings {
    #[serde(default)]
    retries: u32,  // 0 if missing

    #[serde(default = "default_timeout")]
    timeout_ms: u64,
}

fn default_timeout() -> u64 {
    5000
}
```

---

### skip, skip_serializing, skip_deserializing

```rust
#[derive(Serialize, Deserialize)]
struct Session {
    user_id: String,

    #[serde(skip)]
    password_hash: String,

    #[serde(skip_serializing_if = "Option::is_none")]
    nickname: Option<String>,
}
```

`skip` excludes from both directions. The field must
implement `Default` for deserialization to work.

`skip_serializing_if` conditionally omits the field —
commonly used with `Option::is_none` to drop null
fields from JSON.

---

### flatten

Merges a nested struct's fields into the parent:

```rust
#[derive(Serialize, Deserialize)]
struct Pagination {
    page: u32,
    per_page: u32,
}

#[derive(Serialize, Deserialize)]
struct ListUsers {
    query: String,
    #[serde(flatten)]
    pagination: Pagination,
}
// -> {"query": "a", "page": 1, "per_page": 20}
```

**Performance warning:** `flatten` disables serde's
compile-time field lookup optimization. The deserializer
must buffer all keys into a map first. Avoid in
hot paths or high-throughput parsing.

**Collision warning:** If parent and child share a key
name, you get a "duplicate field" error. Check both
structs when debugging.

---

### with / serialize_with / deserialize_with

Delegates serialization to a custom module or function:

```rust
mod iso8601 {
    use chrono::NaiveDate;
    use serde::{self, Deserialize, Deserializer, Serializer};

    const FORMAT: &str = "%Y-%m-%d";

    pub fn serialize<S>(
        date: &NaiveDate,
        serializer: S,
    ) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(
            &date.format(FORMAT).to_string(),
        )
    }

    pub fn deserialize<'de, D>(
        deserializer: D,
    ) -> Result<NaiveDate, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        NaiveDate::parse_from_str(&s, FORMAT)
            .map_err(serde::de::Error::custom)
    }
}

#[derive(Serialize, Deserialize)]
struct Event {
    name: String,
    #[serde(with = "iso8601")]
    date: NaiveDate,
}
```

`with` requires both `serialize` and `deserialize`
functions in the module. Use `serialize_with` or
`deserialize_with` to provide only one direction.

---

### alias

Accepts alternative field names during deserialization:

```rust
#[derive(Deserialize)]
struct Config {
    #[serde(alias = "colour")]
    color: String,
}
// Accepts both {"color": "red"} and {"colour": "red"}
```

Useful for backward compatibility or regional spelling.

---

## Variant Attributes

Applied to enum variants.

### rename (variant)

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "status")]
enum OrderStatus {
    #[serde(rename = "pending_payment")]
    PendingPayment,
    #[serde(rename = "shipped")]
    Shipped,
}
```

---

### alias (variant)

```rust
#[derive(Deserialize)]
enum Color {
    #[serde(alias = "grey")]
    Gray,
    Red,
    Blue,
}
```

---

### skip (variant)

Excludes a variant from serialization and
deserialization. Attempting to serialize a skipped
variant is an error.

---

### other

Catch-all for unknown variant names during
deserialization. Must be a unit variant:

```rust
#[derive(Deserialize)]
#[serde(tag = "type")]
enum Event {
    Click { x: i32, y: i32 },
    Scroll { delta: f64 },
    #[serde(other)]
    Unknown,
}
```

Essential for forward compatibility — new event types
added by the server won't break existing clients.
