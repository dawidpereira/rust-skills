---
name: rust-types
description: >
  Rust type system design — generics, traits, dispatch, newtypes,
  typestate, PhantomData. Use when designing type-safe APIs,
  encountering E0277/E0308/E0599/E0038, choosing between static
  and dynamic dispatch, or implementing sealed traits and
  parse-don't-validate patterns. Also use when working with
  generic bounds, trait objects, newtype wrappers, or making
  invalid states unrepresentable through the type system.
---

# Type System Design

## Core Question

**What invariants should the type system enforce?**

Every time you reach for a `String` where an enum would work,
a `u64` where a `UserId` would be clearer, or a runtime check
where a compile-time guarantee is possible — ask whether the
type system can do the work for you.

The Rust type system is powerful enough to catch entire
categories of bugs at compile time. The question is always:
is the added type complexity worth the guarantee?

---

## Error → Design Question

| Error | Don't Just Say | Ask Instead |
|-------|----------------|-------------|
| E0277 (trait not satisfied) | "Add the bound" | Is the trait design right for this use case? |
| E0308 (type mismatch) | "Cast it" | Are the types modeling the domain correctly? |
| E0599 (method not found) | "Wrong type" | Should this method exist on this type? |
| E0038 (not object-safe) | "Box it" | Should this use static dispatch instead? |

---

## Quick Decisions

| Situation | Reach For | Why |
|-----------|-----------|-----|
| Prevent mixing IDs (user_id vs order_id) | Newtype wrapper | Compile-time distinction, zero cost |
| Value must be validated (email, port) | Parse-don't-validate newtype | Validity guaranteed by construction |
| Mutually exclusive states | Enum | Invalid states unrepresentable |
| Optional value | `Option<T>` | Absence is explicit |
| Fallible operation | `Result<T, E>` | Failure is explicit |
| State machine transitions | Typestate pattern | Invalid transitions won't compile |
| Prevent external trait implementations | Sealed trait | Evolution without breaking changes |
| Add methods to foreign types | Extension trait | Works around orphan rules |
| Type-level marker, no runtime data | `PhantomData<T>` | Zero-size, expresses type relationship |
| Fixed set of valid values (not strings) | Enum, not `&str` | Typos caught at compile time |
| Heterogeneous collection | `Box<dyn Trait>` | Different concrete types, same interface |
| Monomorphized hot path | `impl Trait` / generics | Zero-cost, no vtable dispatch |
| Callback parameter | `impl Fn()` bound | Most flexible for callers, zero-cost |
| Mutable callback | `impl FnMut()` bound | When the callback needs to mutate captured state |
| One-shot callback (ownership) | `impl FnOnce()` bound | Callback consumes captured values |
| Closures that cross threads | `move` + `Send + 'static` | Forces value capture for thread safety |
| Storing mixed closures | `Box<dyn Fn()>` | Type erasure when closure types vary |
| Custom collection iteration | implement `Iterator` + `IntoIterator` | Enables for loops and adapter chains |
| Lazy transformation pipeline | iterator adapters (map, filter, flat_map) | Zero-allocation chains until collect |

---

## Static vs Dynamic Dispatch

```text
Need different concrete types in the same collection?
├── Yes → dyn Trait (dynamic dispatch)
│   └── Box<dyn Trait> for owned, &dyn Trait for borrowed
└── No → impl Trait or generics (static dispatch)
    └── Monomorphized: zero overhead, larger binary
```

| | Static (`impl Trait` / generics) | Dynamic (`dyn Trait`) |
|---|---|---|
| Performance | Inlined, zero overhead | Vtable lookup per call |
| Binary size | Larger (monomorphized copies) | Smaller (single implementation) |
| Heterogeneous collections | No | Yes |
| Compile time | Slower | Faster |
| Object safety required | No | Yes |

**Object safety requires:** no `Self: Sized` bound, no
generic methods, no `Self` in return position
(without `-> Box<dyn Trait>`).

---

## Newtype Pattern

Zero-cost wrapper that adds type safety:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(u64);

// These are now distinct types — can't mix them
fn get_order(user: UserId, order: OrderId) -> Order { ... }
```

### Parse-Don't-Validate

Validate at construction, trust the type afterward:

```rust
pub struct Email(String);

impl Email {
    pub fn new(s: &str) -> Result<Self, EmailError> {
        if s.contains('@') && s.len() > 3 {
            Ok(Email(s.to_string()))
        } else {
            Err(EmailError::Invalid(s.to_string()))
        }
    }

    pub fn as_str(&self) -> &str { &self.0 }
}

// Once you have an Email, it's valid — no re-checking needed
fn send(to: &Email, body: &str) { ... }
```

---

## Usage Scenarios

**Scenario 1:** "I have a function taking two u64 parameters
and callers keep swapping them"
→ Wrap each in a newtype: `UserId(u64)` and `OrderId(u64)`.
Zero runtime cost, compiler catches swaps.

**Scenario 2:** "Should I use `Box<dyn Handler>` or generics
for my plugin system?"
→ If you need a `Vec<Box<dyn Handler>>` with different
concrete types, use dynamic dispatch. If each call site uses
one known type, use `impl Handler` for zero-cost dispatch.

**Scenario 3:** "I'm getting E0277 'the trait bound is not
satisfied' everywhere"
→ Don't just add bounds. Check: is the bound on the right
level (impl block vs struct)? Are you constraining too early?
Bounds on the struct definition propagate everywhere — prefer
bounds on impl blocks.

---

## Reference Files

| File | Read When |
|------|-----------|
| [references/generics.md](references/generics.md) | Working with generic bounds, Into/AsRef/From patterns, avoiding over-abstraction |
| [references/type-driven.md](references/type-driven.md) | Designing newtypes, enums for states, PhantomData, parse-don't-validate, typestate |
| [references/trait-design.md](references/trait-design.md) | Sealed traits, extension traits, object safety, common trait implementations |
| [references/closures-and-iterators.md](references/closures-and-iterators.md) | Fn/FnMut/FnOnce traits, capture semantics, move closures, returning closures, custom Iterator/IntoIterator, FromIterator |

---

## Cross-References

| When | Check |
|------|-------|
| Ownership issues with generic types | rust-ownership → Quick Decisions |
| Error types and Result patterns | rust-errors → Quick Decisions |
| API design beyond type system (builders, naming) | rust-api → Quick Decisions |
| FFI newtypes with #[repr(transparent)] | rust-unsafe → Quick Decisions |
| Move closures for spawn, Send bounds | rust-async → Quick Decisions |
| Iterator chains, lazy evaluation | rust-perf → Quick Decisions |
