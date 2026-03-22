# Anti-Patterns Index

Common Rust anti-patterns. Each row links to the skill or reference that provides full
guidance and code examples.

| Anti-Pattern | Description | Guidance |
|-------------|-------------|----------|
| unwrap abuse | Using `.unwrap()` in production code instead of `?` or `.expect()` with context | rust-error-handling |
| expect without context | Lazy `.expect("failed")` messages that give no debugging information | rust-error-handling |
| excessive cloning | Calling `.clone()` when borrowing (`&T`) would work; wasted allocations | rust-performance |
| lock across await | Holding a `MutexGuard` across an `.await` point, causing deadlocks | rust-async |
| String for &str | Accepting `String` in function params when `&str` suffices | rust-api-design |
| Vec for &[T] | Accepting `Vec<T>` when a slice `&[T]` would work | rust-api-design |
| indexing over iteration | Using `v[i]` in a loop instead of iterators; bounds checks on every access | rust-performance |
| panic for expected errors | Using `panic!` for recoverable error conditions instead of `Result` | rust-error-handling |
| empty error catch | Writing `let _ = fallible()` or `.ok()` to silently discard errors | rust-error-handling |
| over-abstraction | Creating trait hierarchies and generic layers before concrete needs emerge | rust-api-design |
| premature optimization | Optimizing without profiling; sacrificing readability for unmeasured gains | rust-performance |
| type erasure overuse | Using `Box<dyn Trait>` everywhere instead of generics; hidden allocation costs | rust-api-design |
| format! in hot paths | Using `format!()` for string building in performance-critical loops | rust-performance |
| collect intermediate | Collecting into a `Vec` mid-chain instead of keeping the iterator lazy | rust-performance |
| stringly typed | Using `String` where an enum or newtype would enforce valid states at compile time | rust-api-design |
