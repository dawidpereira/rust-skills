# Declarative Macros (macro_rules!)

## Basic Patterns

### Matching Literals

```rust
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
    (to $name:expr) => {
        println!("Hello, {}!", $name);
    };
}

say_hello!();           // Hello!
say_hello!(to "Alice"); // Hello, Alice!
```

### Matching Identifiers

```rust
macro_rules! make_fn {
    ($name:ident, $body:expr) => {
        fn $name() -> i32 {
            $body
        }
    };
}

make_fn!(answer, 42);
assert_eq!(answer(), 42);
```

### Matching Expressions

```rust
macro_rules! check {
    ($cond:expr, $msg:expr) => {
        if !$cond {
            panic!("{}", $msg);
        }
    };
}
```

---

## Fragment Specifier Reference

| Specifier | Matches | Example Input |
| --- | --- | --- |
| `$x:expr` | Expression | `1 + 2`, `foo()`, `{ bar }` |
| `$x:ident` | Identifier | `my_var`, `MyType` |
| `$x:ty` | Type | `i32`, `Vec<String>` |
| `$x:pat` | Pattern | `Some(x)`, `1..=5`, `_` |
| `$x:tt` | Token tree | Any single token or `()`/`[]`/`{}`-delimited group |
| `$x:literal` | Literal | `42`, `"hello"`, `true` |
| `$x:path` | Path | `std::io::Error` |
| `$x:meta` | Attribute content | `derive(Debug)`, `allow(unused)` |
| `$x:item` | Item | `fn foo() {}`, `struct Bar;` |
| `$x:vis` | Visibility | `pub`, `pub(crate)`, *(empty)* |
| `$x:stmt` | Statement | `let x = 1`, `x.push(2)` |
| `$x:block` | Block | `{ x + 1 }` |

**Key rule:** After matching a fragment, only certain
tokens can follow. For example, `$x:expr` can only be
followed by `=>`, `,`, or `;`. This prevents ambiguity.

---

## Repetition

### Generating Match Arms

```rust
macro_rules! enum_str {
    (
        $name:ident { $( $variant:ident ),+ $(,)? }
    ) => {
        impl $name {
            fn as_str(&self) -> &'static str {
                match self {
                    $( Self::$variant => stringify!($variant), )+
                }
            }
        }
    };
}

enum Color { Red, Green, Blue }
enum_str!(Color { Red, Green, Blue });
```

### Implementing Traits for Multiple Types

```rust
macro_rules! impl_display_as_debug {
    ( $( $t:ty ),+ $(,)? ) => {
        $(
            impl std::fmt::Display for $t {
                fn fmt(
                    &self,
                    f: &mut std::fmt::Formatter<'_>,
                ) -> std::fmt::Result {
                    write!(f, "{:?}", self)
                }
            }
        )+
    };
}

impl_display_as_debug!(MyError, MyWarning, MyInfo);
```

### Trailing Comma Handling

Use `$(,)?` to accept an optional trailing comma:

```rust
macro_rules! list {
    ( $( $item:expr ),+ $(,)? ) => {
        vec![ $( $item ),+ ]
    };
}

let a = list![1, 2, 3];   // works
let b = list![1, 2, 3,];  // also works
```

---

## Ordering Rules

Arms are tried top to bottom. Place specific patterns
before general ones:

```rust
macro_rules! describe {
    (0) => { "zero" };
    ($x:literal) => { "some literal" };
    ($x:expr) => { "some expression" };
}
```

If `($x:expr)` came first, it would match everything
and the literal arms would be unreachable.

---

## Scoping and Visibility

### Within a Crate

`macro_rules!` definitions are available after their
definition point in the source file. To use a macro
across modules, define it before the module declarations
or use `#[macro_use]` on the module:

```rust
// In lib.rs — define before modules that use it
macro_rules! helper { () => {}; }

mod a; // can use helper!
mod b; // can use helper!
```

### Across Crates with `#[macro_export]`

```rust
#[macro_export]
macro_rules! public_macro {
    ( $( $item:expr ),* ) => {
        vec![ $( $item ),* ]
    };
}
```

`#[macro_export]` places the macro at the crate root
regardless of where it is defined. Users import it with
`use my_crate::public_macro;`.

### The `$crate` Prefix

Always use `$crate::` to reference items from the
macro's defining crate:

```rust
#[macro_export]
macro_rules! create_thing {
    ($val:expr) => {
        $crate::Thing::new($val)
    };
}
```

This ensures correct resolution whether the macro is
called from inside or outside the crate.

---

## Common Patterns

### Enum Dispatch

```rust
macro_rules! dispatch {
    (
        $self:ident, $method:ident
        ( $( $arg:ident ),* )
        => [ $( $variant:ident ),+ ]
    ) => {
        match $self {
            $(
                Self::$variant(inner) => {
                    inner.$method( $( $arg ),* )
                }
            )+
        }
    };
}

enum Transport { Tcp(TcpStream), Uds(UnixStream) }

impl Transport {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        dispatch!(self, write(buf) => [Tcp, Uds])
    }
}
```

### Newtype Delegation

```rust
macro_rules! delegate {
    (
        $wrapper:ident($inner:ty)
        => $( fn $method:ident(&self $(, $p:ident: $pt:ty )* )
              -> $ret:ty );+ $(;)?
    ) => {
        impl $wrapper {
            $(
                pub fn $method(
                    &self $(, $p: $pt )*
                ) -> $ret {
                    self.0.$method( $( $p ),* )
                }
            )+
        }
    };
}

struct UserId(String);
delegate!(UserId(String) =>
    fn len(&self) -> usize;
    fn is_empty(&self) -> bool
);
```

### Builder Field Generation

```rust
macro_rules! builder_fields {
    (
        $builder:ident {
            $( $field:ident : $ty:ty ),+ $(,)?
        }
    ) => {
        impl $builder {
            $(
                pub fn $field(mut self, val: $ty) -> Self {
                    self.$field = Some(val);
                    self
                }
            )+
        }
    };
}
```

---

## Debugging with `cargo expand`

Install and run:

```bash
cargo install cargo-expand
cargo expand                # expand entire crate
cargo expand module::name   # expand specific module
```

`cargo expand` shows the fully expanded Rust code after
all macros have been processed. This is the primary tool
for diagnosing unexpected macro behavior.

For a quick check without installing `cargo-expand`:

```bash
cargo rustc -- -Zunpretty=expanded  # nightly only
```

### Debugging Strategy

1. Write the code you want the macro to generate by hand
2. Verify the handwritten code compiles
3. Write the macro to produce that exact code
4. Run `cargo expand` and diff against the handwritten
   version

---

## Anti-Patterns

### Macros That Should Be Functions

If a macro only operates on expressions and does not
generate new identifiers or types, it should probably be
a function or generic:

```rust
// Bad: this is just a function
macro_rules! add {
    ($a:expr, $b:expr) => { $a + $b };
}

// Good
fn add(a: i32, b: i32) -> i32 { a + b }
```

### Macros That Hide Control Flow

Macros that contain `return`, `break`, or `continue`
are hard to reason about at call sites:

```rust
// Bad: hidden return
macro_rules! try_or_return {
    ($e:expr) => {
        match $e {
            Ok(v) => v,
            Err(_) => return,
        }
    };
}
```

Use the `?` operator or explicit control flow instead.

### Overly Complex Macros

If a macro has more than 5–6 arms or deep nesting,
consider whether a proc macro would be clearer. Proc
macros have full Rust for logic; `macro_rules!` has
pattern matching only.
