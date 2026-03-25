---
name: rust-macros
description: >
  Rust macros — macro_rules!, proc macros, derive macros,
  attribute macros, fragment specifiers, repetition,
  hygiene, cargo expand, when to use macros vs generics.
  Use when writing or debugging declarative or procedural
  macros, choosing between macros and other abstractions,
  understanding expansion errors, or generating repetitive
  code.
---

# Macros

## Core Question

**Is a macro the simplest way to eliminate this
repetition, or would a function, generic, or trait do
the job?**

Macros generate code at compile time. They are powerful
but harder to read, debug, and maintain than plain Rust.
Reach for them only after ruling out simpler
alternatives.

---

## Error → Design Question

| Symptom | Ask Instead |
| --- | --- |
| "no rules expected the token" | Does your fragment specifier match the input? |
| "unexpected end of macro" | Are repetition delimiters balanced? |
| "macro expanded to expression" | Does the call site expect a statement or expression? |
| "local ambiguity" | Are your macro arms ordered most-specific-first? |
| Proc macro panic | Is your TokenStream handling all input shapes? |
| "recursion limit reached" | Is recursive expansion necessary, or can you restructure? |

---

## Quick Decisions

| Situation | Reach For | Why |
| --- | --- | --- |
| Eliminating repeated struct/enum boilerplate | `macro_rules!` with repetition | Generates variants without proc macro overhead |
| Code generation from attributes | Derive proc macro | Runs at compile time, integrates with `#[derive]` |
| Transforming function signatures | Attribute proc macro | Full control over the annotated item |
| DSL or custom syntax | Function-like proc macro | Arbitrary input parsing via `syn` |
| 2–3 similar match arms | Just duplicate the code | Macro not worth the complexity |
| Generating test cases | `macro_rules!` with test names | Fast to write, easy to extend |
| Conditional compilation | `cfg` attributes, not macros | Built-in, well-understood, no expansion |
| Debugging macro expansion | `cargo expand` | Shows fully expanded code |
| Macro vs generic function | Prefer generic if types differ but logic is same | Generics are type-checked, macros are not |
| Macro vs trait default impl | Prefer trait if behavior varies by type | Trait dispatch is idiomatic Rust |
| Exporting macros from library | `#[macro_export]` for `macro_rules!`, re-export for proc macros | Ensures visibility across crate boundaries |
| Cross-crate proc macros | Separate `-derive` or `-macros` crate | Proc macros must live in their own crate |

---

## Fragment Specifiers

| Specifier | Matches | Use When |
| --- | --- | --- |
| `expr` | Any expression | Values, function calls, blocks |
| `ident` | Identifier | Names for types, functions, variables |
| `ty` | Type | Type parameters, annotations |
| `pat` | Pattern | Match arms, let bindings |
| `tt` | Single token tree | Catch-all, forwarding tokens |
| `literal` | Literal value | Strings, numbers, booleans |
| `path` | Type path | `std::collections::HashMap` |
| `meta` | Attribute contents | `derive(Debug)`, `cfg(test)` |
| `item` | Top-level item | Functions, structs, impls |
| `vis` | Visibility qualifier | `pub`, `pub(crate)`, empty |
| `stmt` | Statement | Let bindings, expressions with `;` |
| `block` | Brace-delimited block | `{ ... }` bodies |

---

## Repetition Patterns

Three repetition operators with an optional separator:

```rust
macro_rules! make_list {
    // Zero or more
    ( $( $item:expr ),* ) => {
        vec![ $( $item ),* ]
    };
}

macro_rules! require_one {
    // One or more
    ( $( $item:expr ),+ ) => {
        vec![ $( $item ),+ ]
    };
}

macro_rules! optional_label {
    ( $( $label:ident : )? $value:expr ) => {
        println!("{}", $value);
    };
}
```

Nested repetition handles parallel lists:

```rust
macro_rules! impl_from {
    ( $target:ident : $( $source:ty => $variant:ident ),+ ) => {
        $(
            impl From<$source> for $target {
                fn from(val: $source) -> Self {
                    Self::$variant(val)
                }
            }
        )+
    };
}
```

---

## The Macro vs Generic Decision

1. **Can a function do it?** → Use a function.
2. **Can a generic do it?** → Use a generic.
3. **Can a trait do it?** → Use a trait.
4. **None of the above?** → Then use a macro.

Macros are the right tool when you need to:
- Generate new identifiers or types
- Repeat code with varying structure (not just types)
- Create syntax that Rust's type system cannot express
- Implement a trait for a list of types

---

## macro_rules! Hygiene

Identifiers inside `macro_rules!` are hygienic — they
do not collide with names at the call site. However,
this means a macro cannot reference local variables from
the caller unless passed in as arguments.

Use `$crate` to reference items from the defining crate:

```rust
#[macro_export]
macro_rules! create_error {
    ( $msg:expr ) => {
        $crate::Error::new($msg)
    };
}
```

Without `$crate`, the macro breaks when used from
another crate because `Error` resolves in the caller's
namespace.

---

## Usage Scenarios

**Scenario 1:** "I have 12 structs that all need the
same trait impl with minor variations"
→ Write a `macro_rules!` with repetition that takes
the struct name and the varying parts as parameters.
See `references/declarative.md` for trait impl patterns.

**Scenario 2:** "I want to generate tests for multiple
input/output pairs"
→ Use `macro_rules!` to generate `#[test]` functions
with unique names from a list of cases.
See `references/declarative.md` for test generation.

**Scenario 3:** "I need a `#[derive(MyTrait)]` that
generates methods based on struct fields"
→ Create a proc macro crate with `syn` and `quote`.
Parse `DeriveInput`, iterate fields, emit impl block.
See `references/procedural.md` for derive macro setup.

---

## Reference Files

| File | Read When |
| --- | --- |
| [references/declarative.md](references/declarative.md) | Writing macro_rules!, fragment specifiers, repetition patterns, common recipes, debugging with cargo expand |
| [references/procedural.md](references/procedural.md) | Derive macros, attribute macros, function-like macros, syn/quote, proc macro testing |

---

## Cross-References

| When | Check |
| --- | --- |
| Generics as alternative to macros | rust-types → Quick Decisions |
| Test generation macros | rust-tests → Quick Decisions |
| cargo expand and lint setup | rust-quality → Quick Decisions |
| Macro naming and documentation | rust-api → Quick Decisions |
