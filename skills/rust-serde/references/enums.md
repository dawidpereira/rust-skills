# Enum Representation

## The Four Strategies

Given this enum:

```rust
#[derive(Serialize, Deserialize)]
enum Shape {
    Circle { radius: f64 },
    Rect { width: f64, height: f64 },
}
```

### Externally Tagged (default)

No attribute needed. The variant name wraps the content:

```json
{ "Circle": { "radius": 5.0 } }
```

- Unambiguous and self-describing.
- Awkward for most JSON APIs — clients expect a flat
  object with a type discriminator, not a wrapper key.
- Works well for Rust-to-Rust communication (bincode,
  postcard).

### Internally Tagged

```rust
#[serde(tag = "type")]
```

```json
{ "type": "Circle", "radius": 5.0 }
```

- The tag field is merged into the variant's content.
- Only works with struct variants and unit variants
  (not tuple variants or newtype variants wrapping
  non-struct types).
- **Best choice for JSON APIs** — matches TypeScript
  discriminated unions and most API conventions.

### Adjacently Tagged

```rust
#[serde(tag = "type", content = "data")]
```

```json
{ "type": "Circle", "data": { "radius": 5.0 } }
```

- Tag and content are separate sibling keys.
- Works with all variant kinds including tuples.
- Good for **versioned APIs** where the content
  structure changes between versions.
- Good for **event systems** where you want to inspect
  the type before parsing the payload.

### Untagged

```rust
#[serde(untagged)]
```

```json
{ "radius": 5.0 }
```

- No discriminator — serde tries each variant in
  declaration order until one succeeds.
- Error messages are terrible: "data did not match
  any variant."
- **Use as a last resort** — only when the format is
  imposed externally and has no type field.

---

## Decision Matrix

| Criterion | Ext. | Int. | Adj. | Untag. |
| --- | --- | --- | --- | --- |
| JSON API standard | no | **yes** | maybe | no |
| TypeScript compat | poor | **good** | good | fragile |
| Tuple variants | yes | no | yes | yes |
| Self-describing | yes | yes | yes | **no** |
| Error quality | good | good | good | **poor** |
| Rust-to-Rust binary | **yes** | no | no | no |
| Inspect type before parsing content | no | no | **yes** | no |

**Start with internally tagged.** Switch to adjacently
tagged if you need tuple variants or lazy content
parsing. Use externally tagged for binary formats. Use
untagged only when forced.

---

## Forward Compatibility with `#[serde(other)]`

When consuming events or messages from an evolving
source, unknown variants will cause deserialization to
fail. Add a catch-all:

```rust
#[derive(Debug, Deserialize)]
#[serde(tag = "type")]
enum WebhookEvent {
    OrderCreated { order_id: String },
    OrderShipped { order_id: String, carrier: String },
    #[serde(other)]
    Unknown,
}
```

`Unknown` absorbs any variant name not listed. This
means your code won't break when the upstream adds new
event types.

**Constraints:**
- `other` must be a unit variant (no fields).
- Only works with internally tagged and adjacently
  tagged enums.
- Does not work with externally tagged or untagged.

---

## Renaming Variants

Use `rename_all` at the enum level or `rename` on
individual variants:

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum Action {
    CreateUser { name: String },
    DeleteUser { user_id: String },
}
// tag values: "create_user", "delete_user"
```

Common convention for JSON APIs is `snake_case` or
`camelCase` variant names:

```rust
#[serde(tag = "type", rename_all = "camelCase")]
```

---

## Nested Enums and Flatten

Flattening an enum field into a parent struct can
create surprising results:

```rust
#[derive(Serialize, Deserialize)]
struct Envelope {
    id: String,
    #[serde(flatten)]
    payload: Message,
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Message {
    Text { body: String },
    Image { url: String },
}
```

This produces:

```json
{ "id": "abc", "type": "Text", "body": "hello" }
```

The `type` discriminator and variant fields are lifted
into the parent object. This is often the desired shape
for API responses.

**Pitfalls:**
- `deny_unknown_fields` on `Envelope` will conflict
  with the flattened enum's fields.
- Performance cost: flatten requires buffering all
  keys into an intermediate map.
- If `Envelope` has a field named `type`, it collides
  with the tag.

---

## Enum Variants with Different Shapes

Internally tagged enums require struct or unit variants.
If you need to mix shapes, use adjacently tagged:

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "op", content = "args")]
enum Command {
    Ping,                           // unit
    Echo(String),                   // newtype
    Move { x: f64, y: f64 },       // struct
    Batch(Vec<Command>),            // newtype
}
```

```json
{ "op": "Ping", "args": null }
{ "op": "Echo", "args": "hello" }
{ "op": "Move", "args": { "x": 1.0, "y": 2.0 } }
```

Adjacently tagged handles all variant kinds because
the content is always in a separate key.

---

## String Enums

For enums with no data, serde serializes as plain
strings by default (externally tagged unit variants):

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
enum Role {
    Admin,
    Editor,
    Viewer,
}
// -> "admin", "editor", "viewer"
```

This works the same for all tagging strategies when
applied to unit-only enums. No tag/content attributes
needed.
