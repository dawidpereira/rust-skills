# Closures and Iterators

## The Three Closure Traits

Every closure implements one or more of `Fn`, `FnMut`,
`FnOnce`. The compiler chooses based on how the closure
uses its captured variables.

```text
FnOnce  ← all closures implement this
  ↑
FnMut   ← closures that don't consume captures
  ↑
Fn      ← closures that don't mutate captures
```

- **`Fn(&self)`** — reads captures by shared reference.
  Callable any number of times, even concurrently.
- **`FnMut(&mut self)`** — mutates captures by exclusive
  reference. Callable multiple times, not concurrently.
- **`FnOnce(self)`** — may move out of captures. Callable
  exactly once.

```rust
let name = String::from("Alice");
let greet = || println!("Hello, {name}");   // Fn
greet(); greet();

let mut count = 0;
let mut inc = || { count += 1; };           // FnMut
inc(); inc();

let consume = || drop(name);                // FnOnce
consume();
// consume();  // Error: already moved
```

---

## Capture Semantics

The compiler captures the minimum required:

| Access in body | Capture mode | Trait |
|----------------|-------------|-------|
| `&x` (read) | By shared ref `&T` | `Fn` |
| `&mut x` (mutate) | By exclusive ref `&mut T` | `FnMut` |
| `drop(x)` or move out | By value `T` | `FnOnce` |

### The `move` Keyword

`move` forces all captures by value. The closure takes
ownership regardless of how variables are used in the
body.

```rust
let name = String::from("Alice");
let greet = move || println!("{name}");
// `name` is no longer accessible here
```

**When `move` is required:**

- Spawning threads or async tasks — closure must outlive
  the current scope
- Returning closures — captured references would dangle
- Sending closures across channels

For `Copy` types, `move` copies the value. The original
remains usable.

---

## Choosing Closure Bounds for Parameters

Accept the weakest bound you need: call once →
`FnOnce` (most permissive), multiple times → `FnMut`,
concurrent → `Fn`. For return types the hierarchy
inverts: `Fn` is most useful to callers.

---

## Returning Closures

```rust
// Concrete return (zero-cost, single closure type)
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |y| x + y
}

// Type-erased return (different closure types per branch)
fn make_op(add: bool) -> Box<dyn Fn(i32) -> i32> {
    if add {
        Box::new(|x| x + 1)
    } else {
        Box::new(|x| x - 1)
    }
}
```

`impl Fn()` requires a single concrete type. Use
`Box<dyn Fn()>` when branches return different closures.

---

## Closures in Structs

### Generic field (monomorphized, zero-cost)

```rust
struct Retry<F: Fn() -> bool> {
    check: F,
    max_attempts: u32,
}

impl<F: Fn() -> bool> Retry<F> {
    fn run(&self) -> bool {
        for _ in 0..self.max_attempts {
            if (self.check)() { return true; }
        }
        false
    }
}
```

Each distinct closure creates a unique `Retry<F>` type.

### Type-erased field (dynamic, heap-allocated)

```rust
struct EventHandler {
    callbacks: Vec<Box<dyn Fn(&Event)>>,
}
```

Use `Box<dyn Fn()>` when the struct must hold closures
of different types.

---

## Closures That Outlive Their Scope

Requires `'static` + `move` so no borrowed references
can dangle. Add `Send` when crossing thread boundaries:

```rust
fn register_callback<F>(f: F)
where
    F: Fn() + Send + 'static,
{
    std::thread::spawn(move || { f(); });
}
```

---

## Common Closure Patterns

### Callback registration

```rust
struct Button {
    on_click: Option<Box<dyn FnMut()>>,
}

impl Button {
    fn set_on_click(
        &mut self,
        cb: impl FnMut() + 'static,
    ) {
        self.on_click = Some(Box::new(cb));
    }
}
```

### Strategy pattern with generics

```rust
struct Processor<F: Fn(&str) -> String> {
    transform: F,
}
```

---

## Fn Trait Decision Table

| Need | Use | Why |
|------|-----|-----|
| Call multiple times, no mutation | `Fn(&self)` | Most permissive for callers |
| Call multiple times, mutates captures | `FnMut(&mut self)` | Allows mutable state |
| Call exactly once, consumes captures | `FnOnce(self)` | Allows move out of captures |
| Store in struct, type known | `F: Fn()` generic | Zero-cost, monomorphized |
| Store in struct, type erased | `Box<dyn Fn()>` | Multiple closure types |
| Return from function | `impl Fn()` | Zero-cost, opaque type |
| Return different closures conditionally | `Box<dyn Fn()>` | Must erase type |

---

## Custom Iterators

### Implementing `Iterator`

Requires one associated type and one method:

```rust
struct Countdown { remaining: u32 }

impl Iterator for Countdown {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.remaining == 0 {
            None
        } else {
            self.remaining -= 1;
            Some(self.remaining + 1)
        }
    }
}
```

### When to Write a Custom Iterator

- Iterating over a custom data structure
- Iteration logic doesn't compose from standard adapters
- You need to yield computed values with internal state

Otherwise, use adapter chains (`map`, `filter`,
`flat_map`).

---

## `IntoIterator`: Enabling For Loops

```rust
struct Team { members: Vec<String> }

impl IntoIterator for Team {
    type Item = String;
    type IntoIter = std::vec::IntoIter<String>;
    fn into_iter(self) -> Self::IntoIter {
        self.members.into_iter()
    }
}

impl<'a> IntoIterator for &'a Team {
    type Item = &'a String;
    type IntoIter = std::slice::Iter<'a, String>;
    fn into_iter(self) -> Self::IntoIter {
        self.members.iter()
    }
}
```

Implement both owned (`for x in team`) and borrowed
(`for x in &team`) variants.

---

## Iterator Adapter Patterns

Lazy, chainable transformations — zero allocation until
collected:

```rust
let names: Vec<String> = users
    .iter()
    .filter(|u| u.active)
    .map(|u| u.name.clone())
    .collect();
```

| Adapter | Purpose |
|---------|---------|
| `map` | Transform each element |
| `filter` | Keep elements matching predicate |
| `flat_map` | Map + flatten nested iterators |
| `take` / `skip` | Limit or offset elements |
| `chain` | Concatenate two iterators |
| `zip` | Pair elements from two iterators |
| `enumerate` | Add index to each element |
| `peekable` | Look ahead without consuming |

---

## `DoubleEndedIterator` and `ExactSizeIterator`

- **`DoubleEndedIterator`** — adds `next_back()`,
  enables `.rev()`. Implement when data supports
  backward traversal.
- **`ExactSizeIterator`** — adds `len()`, enables
  pre-allocation in `collect()`.

---

## Collecting into Custom Types: `FromIterator`

Implement `FromIterator` so `.collect()` produces your
type:

```rust
struct UniqueNames { names: Vec<String> }

impl FromIterator<String> for UniqueNames {
    fn from_iter<I: IntoIterator<Item = String>>(
        iter: I,
    ) -> Self {
        let mut seen = std::collections::HashSet::new();
        let names = iter.into_iter()
            .filter(|n| seen.insert(n.clone()))
            .collect();
        UniqueNames { names }
    }
}
```

---

## Practical Example: Custom Structure Iterator

Separate the collection from its iteration state so
multiple borrows can coexist via `iter()`:

```rust
struct Grid<T> { cells: Vec<T>, width: usize }
struct GridIter<'a, T> { grid: &'a Grid<T>, idx: usize }

impl<T> Grid<T> {
    fn iter(&self) -> GridIter<'_, T> {
        GridIter { grid: self, idx: 0 }
    }
}

impl<'a, T> Iterator for GridIter<'a, T> {
    type Item = (usize, usize, &'a T);

    fn next(&mut self) -> Option<Self::Item> {
        let cells = &self.grid.cells;
        if self.idx >= cells.len() { return None; }
        let (row, col) = (
            self.idx / self.grid.width,
            self.idx % self.grid.width,
        );
        self.idx += 1;
        Some((row, col, &cells[self.idx - 1]))
    }
}
```
