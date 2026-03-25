# Procedural Macros

## Crate Setup

Proc macros must live in a dedicated crate with
`proc-macro = true`. If your main crate is `my_lib`,
create `my_lib_derive` or `my_lib_macros`:

```toml
# my_lib_macros/Cargo.toml
[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

Re-export from the main crate for ergonomic use:

```rust
// my_lib/src/lib.rs
pub use my_lib_macros::MyDerive;
```

---

## Three Kinds of Proc Macros

### Derive Macros

Triggered by `#[derive(MyTrait)]`. Receives the struct
or enum definition, emits additional impl blocks.

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Describe)]
pub fn derive_describe(
    input: TokenStream,
) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;

    let expanded = quote! {
        impl #name {
            pub fn describe() -> &'static str {
                stringify!(#name)
            }
        }
    };

    TokenStream::from(expanded)
}
```

Usage:

```rust
#[derive(Describe)]
struct Order { id: u64, total: f64 }

assert_eq!(Order::describe(), "Order");
```

### Attribute Macros

Applied as `#[my_attr]` on any item. Receives the
attribute arguments and the annotated item. Returns the
replacement item.

```rust
#[proc_macro_attribute]
pub fn log_call(
    attr: TokenStream,
    item: TokenStream,
) -> TokenStream {
    let input = parse_macro_input!(item as syn::ItemFn);
    let name = &input.sig.ident;
    let block = &input.block;
    let sig = &input.sig;
    let attrs = &input.attrs;
    let vis = &input.vis;

    let expanded = quote! {
        #(#attrs)*
        #vis #sig {
            println!("Calling {}", stringify!(#name));
            #block
        }
    };

    TokenStream::from(expanded)
}
```

Usage:

```rust
#[log_call]
fn process(data: &str) -> Result<(), Error> {
    // ...
}
```

### Function-Like Macros

Invoked as `my_macro!(...)` with arbitrary input tokens.
Useful for DSLs or custom syntax.

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    let query = input.to_string();
    let expanded = quote! {
        sqlx::query(#query)
    };
    TokenStream::from(expanded)
}
```

Usage:

```rust
let q = sql!(SELECT * FROM users WHERE id = ?);
```

---

## Parsing with `syn`

### DeriveInput Fields

For derive macros, `DeriveInput` gives you:

```rust
let input = parse_macro_input!(input as DeriveInput);

let name = &input.ident;       // struct/enum name
let generics = &input.generics; // generic params
let data = &input.data;         // fields or variants
```

### Iterating Struct Fields

```rust
fn field_names(
    data: &syn::Data,
) -> Vec<&syn::Ident> {
    match data {
        syn::Data::Struct(s) => {
            s.fields.iter()
                .filter_map(|f| f.ident.as_ref())
                .collect()
        }
        _ => panic!("expected a struct"),
    }
}
```

### Handling Generics

Use `split_for_impl` to correctly place generic
parameters in the generated impl:

```rust
let (impl_generics, ty_generics, where_clause) =
    input.generics.split_for_impl();

let expanded = quote! {
    impl #impl_generics MyTrait for #name #ty_generics
    #where_clause
    {
        fn method(&self) -> String {
            String::from(stringify!(#name))
        }
    }
};
```

---

## Error Handling in Proc Macros

### `compile_error!` for Simple Cases

```rust
if input.generics.type_params().count() > 0 {
    return quote! {
        compile_error!(
            "Describe does not support generic types"
        );
    }
    .into();
}
```

### `syn::Error` for Span-Accurate Diagnostics

```rust
use syn::spanned::Spanned;

fn validate(input: &DeriveInput) -> syn::Result<()> {
    match &input.data {
        syn::Data::Struct(_) => Ok(()),
        _ => Err(syn::Error::new(
            input.ident.span(),
            "Describe can only be derived for structs",
        )),
    }
}

#[proc_macro_derive(Describe)]
pub fn derive_describe(
    input: TokenStream,
) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    match validate(&input) {
        Ok(()) => generate(&input).into(),
        Err(e) => e.to_compile_error().into(),
    }
}
```

`syn::Error` points the compiler error at the exact
source location, making diagnostics much clearer than
`panic!`.

---

## Testing Proc Macros

### Compile-Fail Tests with `trybuild`

```toml
# In the proc macro crate's Cargo.toml
[dev-dependencies]
trybuild = "1"
```

```rust
#[test]
fn compile_tests() {
    let t = trybuild::TestCases::new();
    t.pass("tests/pass/*.rs");
    t.compile_fail("tests/fail/*.rs");
}
```

Create test files with expected errors:

```rust
// tests/fail/not_a_struct.rs
use my_lib_macros::Describe;

#[derive(Describe)]
enum Bad { A, B }

fn main() {}
```

```text
// tests/fail/not_a_struct.stderr
error: Describe can only be derived for structs
```

### Runtime Tests

Test the generated code by using the macro in regular
test functions:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[derive(Describe)]
    struct TestStruct { x: i32 }

    #[test]
    fn describe_returns_name() {
        assert_eq!(TestStruct::describe(), "TestStruct");
    }
}
```

---

## When NOT to Use Proc Macros

| Situation | Better Alternative |
| --- | --- |
| Simple repetition over a list | `macro_rules!` |
| Logic that only varies by type | Generics |
| Behavior that varies by type | Trait with default methods |
| Build-time code generation | `build.rs` script |

Proc macros add compile-time cost because they run
the Rust compiler on the macro crate itself. Every
dependency of the proc macro crate is compiled before
any user code.

Debugging is harder: `cargo expand` shows the output,
but stepping through proc macro logic requires
`eprintln!` or a debugger attached to the compiler
process.

Use proc macros when:
- You need to inspect struct/enum field names and types
- You need to generate code based on attributes
- The input is complex syntax a `macro_rules!` pattern
  cannot express

---

## Example: Complete Derive Macro

A `Builder` derive macro that generates a builder
struct with setter methods:

```rust
use proc_macro::TokenStream;
use quote::{format_ident, quote};
use syn::{parse_macro_input, Data, DeriveInput, Fields};

#[proc_macro_derive(Builder)]
pub fn derive_builder(
    input: TokenStream,
) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    let builder_name = format_ident!("{}Builder", name);

    let fields = match &input.data {
        Data::Struct(s) => match &s.fields {
            Fields::Named(f) => &f.named,
            _ => panic!("Builder requires named fields"),
        },
        _ => panic!("Builder can only be derived for structs"),
    };

    let builder_fields = fields.iter().map(|f| {
        let name = &f.ident;
        let ty = &f.ty;
        quote! { #name: Option<#ty> }
    });

    let setters = fields.iter().map(|f| {
        let name = &f.ident;
        let ty = &f.ty;
        quote! {
            pub fn #name(mut self, val: #ty) -> Self {
                self.#name = Some(val);
                self
            }
        }
    });

    let build_fields = fields.iter().map(|f| {
        let name = &f.ident;
        quote! {
            #name: self.#name.ok_or(
                concat!(
                    stringify!(#name),
                    " is not set"
                )
            )?
        }
    });

    let none_fields = fields.iter().map(|f| {
        let name = &f.ident;
        quote! { #name: None }
    });

    let expanded = quote! {
        pub struct #builder_name {
            #( #builder_fields, )*
        }

        impl #name {
            pub fn builder() -> #builder_name {
                #builder_name {
                    #( #none_fields, )*
                }
            }
        }

        impl #builder_name {
            #( #setters )*

            pub fn build(
                self,
            ) -> Result<#name, &'static str> {
                Ok(#name {
                    #( #build_fields, )*
                })
            }
        }
    };

    TokenStream::from(expanded)
}
```

Usage:

```rust
#[derive(Builder)]
struct Config {
    host: String,
    port: u16,
    workers: usize,
}

let config = Config::builder()
    .host("localhost".into())
    .port(8080)
    .workers(4)
    .build()
    .expect("all fields set");
```
