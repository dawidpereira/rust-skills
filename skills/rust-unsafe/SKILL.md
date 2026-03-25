---
name: rust-unsafe
description: >
  Unsafe Rust and FFI patterns. Use when writing or reviewing unsafe blocks, doing FFI/extern
  calls, working with raw pointers, transmute, MaybeUninit, or #[repr]. Covers SAFETY
  documentation requirements and review checklists. Also use when auditing unsafe code or
  deciding whether unsafe is actually needed.
---

# Unsafe Rust & FFI

## Core Question

**Can this be done without unsafe? If not, what invariants must be upheld?**

Most Rust code never needs `unsafe`. Before reaching for it:

- Check if a safe abstraction already exists in std or a
  well-audited crate.
- Check if the borrow checker complaint reveals a design
  problem, not a limitation.
- If `unsafe` is truly needed, identify every invariant the
  compiler can no longer enforce.

If you cannot precisely state the invariants, you do not
understand the code well enough to use `unsafe`.

---

## Quick Decisions

| Scenario | Unsafe Needed? | Use Instead |
|----------|---------------|-------------|
| Bounds-checked indexing is too slow | Maybe | Prove bounds first, then `get_unchecked` |
| Calling a C library function | Yes | Wrap in safe API with validity checks |
| Sharing data between threads | No | `Arc<Mutex<T>>`, `crossbeam`, channels |
| Reinterpreting bytes as a type | Usually | `bytemuck`, `zerocopy`, or `TryFrom` |
| Global mutable state | No | `OnceLock`, `AtomicT`, `Mutex` |
| Implementing a custom collection | Yes | Encapsulate unsafe behind safe public API |
| Bypassing the borrow checker | No | Fix the ownership design |
| Self-referential struct | No | `pin-project`, `ouroboros`, or redesign |
| SIMD intrinsics | Yes | Use `std::simd` (portable) when possible |
| Inline assembly | Yes | Wrap with safe function, document constraints |

---

## Review Checklist

Before approving any `unsafe` block, verify each applicable
item:

| Category | Check |
|----------|-------|
| **Pointer validity** | Non-null, points to allocated memory, not yet freed |
| **Alignment** | Pointer aligned to `align_of::<T>()` |
| **Initialization** | Memory contains a valid value of type `T` |
| **Aliasing** | No `&mut` and `&` to same memory simultaneously |
| **Lifetime** | Pointed-to data outlives the reference created from it |
| **Send/Sync** | Manual impls are sound — type is actually thread-safe |
| **Uninitialized memory** | Only accessed through `MaybeUninit`, never read before init |
| **FFI signatures** | Match the C header exactly (types, calling convention, nullability) |
| **Drop** | No double-free, no use-after-free, destructors run correctly |
| **Size/layout** | `transmute` source and target have identical size and valid bit patterns |

---

## Required Documentation

### Every unsafe block: `// SAFETY:` comment

```rust
pub fn get(&self, index: usize) -> Option<&T> {
    if index < self.len {
        // SAFETY: bounds check above guarantees index < len,
        // so this access is within the allocated region.
        Some(unsafe { &*self.ptr.as_ptr().add(index) })
    } else {
        None
    }
}
```

### Every unsafe function: `/// # Safety` doc section

```rust
/// Constructs a `Name` from raw UTF-8 bytes without validation.
///
/// # Safety
///
/// - `bytes` must contain valid UTF-8.
/// - `bytes` must not be empty.
/// - The caller retains no mutable reference to `bytes` after this call.
pub unsafe fn from_utf8_unchecked(bytes: Vec<u8>) -> Name {
    Name(String::from_utf8_unchecked(bytes))
}
```

### Every unsafe trait impl: `// SAFETY:` comment above impl

```rust
// SAFETY: All fields of Buffer are Send-safe (Vec<u8> and usize).
// The raw pointer field is only dereferenced while &self is held.
unsafe impl Send for Buffer {}
```

Enforce with: `#![warn(clippy::undocumented_unsafe_blocks)]`

---

## Deprecated → Better

| Deprecated / Risky | Use Instead | Why |
|--------------------|-------------|-----|
| `mem::uninitialized()` | `MaybeUninit<T>` | UB if `T` has invalid bit patterns |
| `mem::zeroed()` for refs/bools | `MaybeUninit<T>` | Null ref and `0u8` bool are instant UB |
| `static mut` | `AtomicT`, `Mutex`, `OnceLock` | Data races are UB even on single core |
| `transmute` for byte reinterpretation | `from_ne_bytes`, `bytemuck::cast` | Checked size/alignment, no surprise UB |
| `transmute` for enum conversion | `TryFrom` impl | Invalid discriminant is UB |
| Raw pointer arithmetic (`ptr + n`) | `ptr::add(n)`, `ptr::offset(n)` | Overflow and alignment are checked in debug |
| `CString::new(s).unwrap().as_ptr()` | Bind `CString` to a variable first | Temporary is dropped, pointer dangles |
| Manual `extern "C"` declarations | `bindgen` | Avoids signature mismatches |

---

## FFI Patterns

### repr attributes

| Attribute | Use When |
|-----------|----------|
| `#[repr(C)]` | Struct passed to/from C code |
| `#[repr(transparent)]` | Newtype wrapper used in FFI (same ABI as inner type) |
| `#[repr(u8)]` / `#[repr(c_int)]` | Enum matching a C enum's underlying type |
| `#[repr(packed)]` | Matching a packed C struct (avoid — causes unaligned access) |

### Safe wrapper pattern

```rust
mod ffi {
    use std::os::raw::c_int;

    extern "C" {
        pub fn compress(src: *const u8, src_len: usize,
                        dst: *mut u8, dst_len: *mut usize) -> c_int;
    }
}

pub fn compress(input: &[u8]) -> Result<Vec<u8>, CompressError> {
    let mut buf = vec![0u8; max_compressed_size(input.len())];
    let mut out_len = buf.len();

    // SAFETY: input and buf are valid slices, out_len points to a valid usize.
    // compress() writes at most out_len bytes to dst.
    let rc = unsafe {
        ffi::compress(
            input.as_ptr(), input.len(),
            buf.as_mut_ptr(), &mut out_len,
        )
    };

    if rc != 0 {
        return Err(CompressError(rc));
    }
    buf.truncate(out_len);
    Ok(buf)
}
```

---

## Usage Scenarios

**Scenario 1:** "I need to call a C library from Rust."
→ Use `bindgen` to generate bindings. Wrap every FFI call
in a safe function that validates inputs and checks return
codes. Use `#[repr(C)]` on structs shared across the
boundary. Use `#[repr(transparent)]` for newtype wrappers
of C types. Bind `CString` to a variable before calling
`.as_ptr()`.

**Scenario 2:** "I'm reviewing code that uses `unsafe` —
what do I check?"
→ Load
[references/checklist.md](references/checklist.md).
For each `unsafe` block: verify a `// SAFETY:` comment
exists and is accurate, check every item in the review
checklist above, and confirm there is no safe alternative.
For `unsafe fn`, verify the `# Safety` doc section lists
all caller obligations.

**Scenario 3:** "The borrow checker won't let me do X —
should I use unsafe?"
→ Almost certainly not. Unsafe does not disable the borrow
checker's rules — it merely hides UB. Redesign ownership
(see rust-ownership), use `RefCell` for interior
mutability, or use `Pin` for self-referential types.

---

## Reference Files

| File | Read When |
|------|-----------|
| [references/checklist.md](references/checklist.md) | Reviewing or auditing unsafe code, FFI bindings, or unsafe trait impls |

---

## Cross-References

| When | Check |
|------|-------|
| Ownership questions in unsafe code | rust-ownership → Quick Decisions |
| Send/Sync trait bounds for async + unsafe | rust-async → Quick Decisions |
| Newtype wrappers for FFI types | rust-types → Quick Decisions |
| Documenting # Safety sections | rust-api → Quick Decisions |
| Clippy unsafe lints | rust-quality → Quick Decisions |
