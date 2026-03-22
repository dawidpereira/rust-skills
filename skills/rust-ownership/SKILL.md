---
name: rust-ownership
description: >
  Rust ownership, borrowing, and lifetime patterns. Use this skill whenever working with
  move semantics, references, smart pointers (Arc, Rc, Box, Cell, RefCell, Mutex, RwLock),
  interior mutability, or encountering E0382/E0597/E0506/E0507/E0515/E0716/E0106 errors.
  Also use when deciding between clone, borrow, or shared ownership, or when lifetime
  annotations are confusing.
---

# Ownership & Lifetimes

## Core Question

**Who should own this data, and for how long?**

Before fixing an ownership error, understand the data's
role:

- Is it shared or exclusive?
- Is it short-lived or long-lived?
- Is it transformed or just read?

The compiler tells you *what* broke. Your job is to figure
out *why* the ownership design led here.

---

## Error → Design Question

Don't just silence the compiler — ask what the error
reveals about your design.

| Error | Don't Just Say | Ask Instead |
|-------|----------------|-------------|
| E0382 (moved value) | "Clone it" | Who should own this data? |
| E0597 (dangling ref) | "Extend lifetime" | Is the scope boundary correct? |
| E0506 (assign while borrowed) | "End borrow first" | Should mutation happen elsewhere? |
| E0507 (move from reference) | "Clone before move" | Why are we moving from a reference? |
| E0515 (return local ref) | "Return owned" | Should the caller own the data? |
| E0716 (temporary dropped) | "Bind to variable" | Why is this a temporary? |
| E0106 (missing lifetime) | "Add 'a" | What is the actual lifetime relationship? |

If you've tried the same fix twice and it cascades, the
ownership *design* is wrong — not just the syntax.

---

## Quick Decisions

| Situation | Reach For | Why |
|-----------|-----------|-----|
| Caller doesn't need data afterward | Move | Zero cost, transfers ownership |
| Read-only access | `&T` | Zero cost borrow |
| Need to modify borrowed data | `&mut T` | Exclusive borrow, zero cost |
| Actually need a separate copy | `.clone()` | Heap allocation — make it explicit |
| Small trivial type (≤16 bytes) | `Copy` | Implicit, free duplication |
| Might need to modify borrowed data | `Cow<'a, T>` | Allocates only if mutated |
| Shared ownership, single thread | `Rc<T>` | Reference counted, no atomics |
| Shared ownership, multi thread | `Arc<T>` | Atomic reference counted |
| Need mutation through `&self` (single thread) | `RefCell<T>` | Runtime borrow checking |
| Need mutation through `&self` (multi thread) | `Mutex<T>` / `RwLock<T>` | Locking, thread-safe |
| Large type passed by value | `Box<T>` | Move costs 8 bytes instead of N |

---

## Smart Pointer Decision Tree

```text
Need shared ownership?
├── Yes → Across threads?
│   ├── Yes → Arc<T>
│   │   └── Need mutation? → Arc<Mutex<T>> or Arc<RwLock<T>>
│   └── No → Rc<T>
│       └── Need mutation? → Rc<RefCell<T>>
└── No → Need heap allocation?
    ├── Yes → Box<T>
    └── No → Use stack (move or borrow)
```

**When choosing between Mutex and RwLock:**
- Reads dominate (>80% reads) → `RwLock<T>`
- Frequent writes or very brief locks → `Mutex<T>`
- Single thread → `RefCell<T>` (no locking overhead)
- Performance critical → consider `parking_lot` crate

---

## Lifetime Elision

Rust elides lifetimes automatically in most cases. Don't
annotate unless required.

**The three elision rules:**
1. Each input reference gets its own lifetime
2. If exactly one input lifetime, output gets that lifetime
3. If `&self` or `&mut self` is an input, output gets `self`'s lifetime

**When you MUST annotate:**
- Multiple input references with ambiguous output lifetime
- Structs holding references
- Multiple distinct lifetime relationships
- `'static` bounds

Use anonymous lifetime `'_` for clarity when the compiler
needs a hint but the specific lifetime doesn't matter:
`impl fmt::Display for Wrapper<'_>`.

---

## Usage Scenarios

**Scenario 1:** "I'm getting E0382 — value used after
being moved"
→ Don't immediately add `.clone()`. Ask: does the second
use actually need ownership? If it only reads, take a
reference. If both uses need ownership, consider `Rc`/`Arc`
or restructuring so one use happens before the other.

**Scenario 2:** "Should I use Arc or clone this config
into each thread?"
→ If the config is read-only after creation,
`Arc<Config>` is cheaper — one allocation shared by all
threads. If each thread might modify its copy, clone
before spawning.

**Scenario 3:** "I need to update a cache behind a shared
reference"
→ This is interior mutability. Single thread?
`RefCell<HashMap<K, V>>`. Multi-thread?
`Mutex<HashMap<K, V>>` or `RwLock<HashMap<K, V>>` if reads
dominate. Consider `dashmap` for concurrent maps.

---

## Reference Files

Read these when you need depth beyond the quick reference above.

| File | Read When |
|------|-----------|
| [references/borrowing.md](references/borrowing.md) | Deciding between borrow, clone, move, Cow; accepting slices vs owned types; lifetime patterns |
| [references/smart-pointers.md](references/smart-pointers.md) | Choosing between Box, Rc, Arc; understanding reference counting; boxing large enum variants |
| [references/interior-mutability.md](references/interior-mutability.md) | Working with RefCell, Mutex, RwLock; mutation through shared references; lock safety in async |

---

## Cross-References

| When | Check |
|------|-------|
| Error types and propagation | rust-errors → Quick Decisions |
| Trait bounds causing ownership issues | rust-types → Quick Decisions |
| Locks held across `.await` | rust-async → Quick Decisions |
| Large enum variants or allocation optimization | rust-perf → Quick Decisions |
