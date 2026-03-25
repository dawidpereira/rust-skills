# Unsafe Code Review Checklist

## Per-Operation Safety Requirements

### Raw Pointer Dereference (`*ptr`)

| Requirement | How to Verify |
|-------------|---------------|
| Non-null | Check for null before deref, or use `NonNull<T>` |
| Aligned to `align_of::<T>()` | Verify source allocation or use `read_unaligned` |
| Points to initialized `T` | Trace data flow — was the memory written before read? |
| No aliasing violation | No `&mut T` exists while `&T` is live (or vice versa) |
| Memory is still allocated | Ensure owner has not dropped or freed |

**Bad:**

```rust
fn first(ptr: *const u32) -> u32 {
    unsafe { *ptr }
}
```

**Good:**

```rust
fn first(ptr: *const u32) -> Option<u32> {
    if ptr.is_null() {
        return None;
    }
    // SAFETY: caller guarantees ptr is aligned, initialized, and valid.
    // Null case handled above.
    Some(unsafe { ptr.read() })
}
```

### Transmute

| Requirement | How to Verify |
|-------------|---------------|
| Source and target are same size | `size_of::<Src>() == size_of::<Dst>()` |
| Bit pattern is valid for target type | No invalid enum discriminants, no null refs, no non-UTF-8 in `str` |
| Alignment is compatible | Check `align_of` for both types |

**Bad:**

```rust
let val: u8 = 2;
let b: bool = unsafe { std::mem::transmute(val) };
```

**Good:**

```rust
let val: u8 = 2;
let b: bool = val != 0;
```

**Bad:**

```rust
let n: u32 = unsafe { std::mem::transmute(some_enum_value) };
```

**Good:**

```rust
#[repr(u32)]
enum Status { Active = 0, Inactive = 1 }

impl TryFrom<u32> for Status {
    type Error = InvalidStatus;
    fn try_from(v: u32) -> Result<Self, Self::Error> {
        match v {
            0 => Ok(Status::Active),
            1 => Ok(Status::Inactive),
            _ => Err(InvalidStatus(v)),
        }
    }
}
```

### Extern / FFI Calls

| Requirement | How to Verify |
|-------------|---------------|
| Signature matches C header exactly | Cross-reference header or use `bindgen` |
| Calling convention correct | `extern "C"` for C functions, `extern "system"` for Win32 |
| Pointer arguments valid for duration of call | No temporaries dropped mid-call |
| Return values checked | C functions signal errors via return codes |
| Strings are null-terminated | Use `CString`, bind to variable before `.as_ptr()` |

**Bad:**

```rust
extern "C" { fn process(s: *const u8); }

fn call(input: &str) {
    unsafe { process(CString::new(input).unwrap().as_ptr() as *const u8); }
}
```

**Good:**

```rust
extern "C" { fn process(s: *const std::os::raw::c_char); }

fn call(input: &str) -> Result<(), NulError> {
    let c_input = CString::new(input)?;
    // SAFETY: c_input is valid for the duration of this call,
    // process() does not store the pointer.
    unsafe { process(c_input.as_ptr()); }
    Ok(())
}
```

### Union Field Access

| Requirement | How to Verify |
|-------------|---------------|
| Active field is known | Track which variant was last written |
| Bit pattern is valid for accessed field | Ensure written type is compatible |
| No drop types without `ManuallyDrop` | Union fields with drop glue are unsound without it |

### Mutable Statics (`static mut`)

| Requirement | How to Verify |
|-------------|---------------|
| No concurrent access | **Hard to guarantee** — prefer `AtomicT`, `Mutex`, `OnceLock` |
| Synchronized if multi-threaded | Use proper atomic ordering or lock |

Prefer elimination over verification — replace `static mut`
with safe alternatives.

---

## FFI Type Patterns

### repr attributes

| C Type | Rust Equivalent |
|--------|----------------|
| `struct` with named fields | `#[repr(C)] struct` |
| Opaque pointer | `#[repr(C)] struct Opaque { _priv: [u8; 0] }` or `extern type` (nightly) |
| Enum with known underlying type | `#[repr(c_int)]` or `#[repr(u32)]` matching the C definition |
| Newtype handle (fd, HANDLE) | `#[repr(transparent)] struct Fd(c_int)` |
| Bitflags | `bitflags!` crate with `#[repr(transparent)]` |

### CString lifetime trap

```rust
// Bad: CString is dropped, pointer dangles
let ptr = CString::new("hello").unwrap().as_ptr();
unsafe { use_ptr(ptr); }

// Good: CString lives long enough
let owned = CString::new("hello").unwrap();
unsafe { use_ptr(owned.as_ptr()); }
```

---

## Safe Wrapper Patterns

### Encapsulate unsafe behind a safe API

1. Validate all inputs in the safe function.
2. Perform the unsafe operation.
3. Validate outputs before returning.
4. Document the `// SAFETY:` justification at each unsafe
   block.

```rust
pub struct AlignedBuffer {
    ptr: NonNull<u8>,
    len: usize,
    layout: Layout,
}

impl AlignedBuffer {
    pub fn new(size: usize, align: usize) -> Result<Self, LayoutError> {
        let layout = Layout::from_size_align(size, align)?;
        // SAFETY: layout has non-zero size (checked by from_size_align).
        let ptr = unsafe { NonNull::new(alloc::alloc(layout)) }
            .ok_or_else(|| LayoutError)?;
        Ok(Self { ptr, len: size, layout })
    }
}

impl Drop for AlignedBuffer {
    fn drop(&mut self) {
        // SAFETY: ptr was allocated with this layout in new().
        unsafe { alloc::dealloc(self.ptr.as_ptr(), self.layout); }
    }
}
```

### Minimize unsafe surface area

- Keep unsafe blocks as small as possible.
- Never put control flow (`if`, `match`, `loop`) inside
  `unsafe {}` unless necessary.
- Extract the unsafe operation into a helper and call it
  from safe code.

---

## Documentation Requirements

### `// SAFETY:` on every unsafe block

Explain **why** the invariants hold, not **what** the
operation does.

**Bad:**

```rust
// SAFETY: we dereference the pointer
unsafe { *ptr }
```

**Good:**

```rust
// SAFETY: ptr is derived from a &[u8] with bounds checked at line 42,
// so it is non-null, aligned, and points to initialized memory.
unsafe { *ptr }
```

### `/// # Safety` on every unsafe function

List every obligation the caller must uphold. Use bullet
points.

### `// SAFETY:` on every unsafe trait impl

State why the implementing type satisfies the trait's
contract.

---

## Review Process

When auditing unsafe code, ask these questions in order:

1. **Is unsafe necessary?** Search for safe alternatives
   first.
2. **Is the unsafe block minimal?** Only the operation that
   requires unsafe should be inside the block.
3. **Is every invariant documented?** Each `// SAFETY:`
   comment must justify why UB cannot occur.
4. **Are the stated invariants correct?** Trace data flow
   to verify claims.
5. **What happens on panic?** Ensure no partially-initialized
   state leaks if a panic unwinds through unsafe code.
6. **Is Send/Sync correct?** If the type has manual
   `Send`/`Sync` impls, verify actual thread safety.
7. **Does Miri pass?** Run `cargo +nightly miri test` to
   detect UB in tests.
8. **Is there a test?** Unsafe code needs tests covering
   edge cases: null, zero-length, max-size, alignment
   boundaries.
