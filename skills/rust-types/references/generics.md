# Generics & Conversion Traits

## Trait Bounds: Only Where Needed

Place bounds on impl blocks, not struct definitions. This
keeps the type usable in contexts where the bound isn't
required.

```rust
// Bad: forces T: Clone everywhere, even when not cloning
struct Cache<T: Clone> {
    data: Vec<T>,
}

// Good: bound only where actually needed
struct Cache<T> {
    data: Vec<T>,
}

impl<T: Clone> Cache<T> {
    fn snapshot(&self) -> Vec<T> {
        self.data.clone()
    }
}
```

### Where Clauses for Readability

```rust
// Hard to read
fn process<T: Serialize + DeserializeOwned + Clone + Debug, E: Error + Send + Sync>(
    data: T,
) -> Result<T, E> { ... }

// Clearer
fn process<T, E>(data: T) -> Result<T, E>
where
    T: Serialize + DeserializeOwned + Clone + Debug,
    E: Error + Send + Sync,
{ ... }
```

### Conditional Implementations

Implement traits for your type only when the inner type
supports it:

```rust
struct Wrapper<T>(T);

impl<T: Clone> Clone for Wrapper<T> {
    fn clone(&self) -> Self {
        Wrapper(self.0.clone())
    }
}

impl<T: Debug> Debug for Wrapper<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        self.0.fmt(f)
    }
}
```

---

## From and Into

**Implement `From<T>`, not `Into<U>`** — the standard library
provides a blanket impl that gives you `Into` for free:

```rust
// Good: implement From
impl From<UserId> for u64 {
    fn from(id: UserId) -> Self {
        id.0
    }
}
// Now UserId automatically implements Into<u64>

// Bad: implementing Into directly
impl Into<u64> for UserId { ... }  // Don't do this
```

### TryFrom for Fallible Conversions

```rust
impl TryFrom<i64> for PositiveInt {
    type Error = InvalidValueError;

    fn try_from(value: i64) -> Result<Self, Self::Error> {
        if value > 0 {
            Ok(PositiveInt(value))
        } else {
            Err(InvalidValueError(value))
        }
    }
}
```

---

## impl Into<T> for Flexible APIs

Accept `impl Into<T>` when callers might have different
source types:

```rust
// Flexible: accepts &str, String, Cow<str>
pub fn set_name(&mut self, name: impl Into<String>) {
    self.name = name.into();
}

// Callers don't need .to_string() or .into()
config.set_name("literal");
config.set_name(computed_string);
```

### When NOT to Use impl Into

- **Trait objects**: `dyn FnMut(impl Into<String>)` doesn't
  work
- **Recursive types**: can cause infinite type resolution
- **Hot paths**: the conversion might allocate — make cost
  explicit

---

## AsRef for Borrowed Access

Use `AsRef<T>` when you only need to borrow, not take
ownership:

```rust
// Accepts &str, String, &String, Cow<str>
fn contains_hello(text: impl AsRef<str>) -> bool {
    text.as_ref().contains("hello")
}

// Accepts &Path, PathBuf, &str, String
fn file_exists(path: impl AsRef<Path>) -> bool {
    path.as_ref().exists()
}
```

### AsRef vs Into vs Borrow

| Trait | Purpose | Allocation |
|-------|---------|-----------|
| `AsRef<T>` | Cheap borrowed view | Never |
| `Into<T>` | Ownership transfer | May allocate |
| `Borrow<T>` | Same as `AsRef` + Eq/Hash/Ord consistency | Never |

Use `Borrow` for HashMap/HashSet keys. Use `AsRef` for
everything else.

---

## Avoiding Over-Abstraction

Generics have costs: compile time, binary size, cognitive
load. Don't abstract prematurely.

**Signs of over-abstraction:**

- Generic has only one concrete implementation
- Type parameter soup: `fn process<T, U, V, W>(...)` with
  4+ type params
- Marker traits with no methods
- Bounds 3+ levels deep: `T: AsRef<U>` where `U: Into<V>`
  where `V: ...`

**The Rule of Three:** Wait for three similar implementations
before abstracting into a generic.

```rust
// Bad: one implementation, no need for generic
fn process<T: AsRef<str>>(input: T) -> String {
    input.as_ref().to_uppercase()  // Just accept &str
}

// Good: concrete when only one type is used
fn process(input: &str) -> String {
    input.to_uppercase()
}
```

Prefer concrete types in private/internal code. Save generics
for public API boundaries where flexibility matters.

---

## impl Trait vs Box<dyn Trait>

Don't use `Box<dyn Trait>` when `impl Trait` works — type
erasure adds heap allocation and vtable dispatch.

```rust
// Bad: unnecessary heap allocation and dynamic dispatch
fn make_iter() -> Box<dyn Iterator<Item = i32>> {
    Box::new((0..10).filter(|x| x % 2 == 0))
}

// Good: zero-cost, monomorphized
fn make_iter() -> impl Iterator<Item = i32> {
    (0..10).filter(|x| x % 2 == 0)
}
```

**When `Box<dyn Trait>` IS right:**

- Heterogeneous collections: `Vec<Box<dyn Handler>>`
- Return type depends on runtime condition
- Recursive types
- Reducing compile time in large codebases

| | `impl Trait` | `Box<dyn Trait>` |
|---|---|---|
| Allocation | Stack | Heap |
| Dispatch | Static (inlined) | Dynamic (vtable) |
| Binary size | Larger (monomorphized) | Smaller |
| Collections | Homogeneous only | Heterogeneous |
