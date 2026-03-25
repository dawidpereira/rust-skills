# Async Streams

## The Stream Trait

`Stream` is the async equivalent of `Iterator`. Where
`Iterator::next()` returns `Option<T>`, `Stream::poll_next()`
returns `Poll<Option<T>>` — yielding items asynchronously.

```toml
[dependencies]
futures = "0.3"
tokio-stream = "0.1"
```

`tokio_stream` re-exports the `Stream` trait and provides
`StreamExt` with combinators tailored for Tokio. `futures`
provides the core trait and additional utilities.

---

## StreamExt Combinators

```rust
use tokio_stream::StreamExt;

let mut stream = tokio_stream::iter(vec![1, 2, 3, 4, 5]);

// map — transform each item
let doubled = stream.map(|x| x * 2);

// filter — keep items matching predicate
let evens = stream.filter(|x| x % 2 == 0);

// filter_map — filter and transform in one step
let parsed = string_stream.filter_map(|s| s.parse::<i32>().ok());

// take — stop after N items
let first_three = stream.take(3);

// take_while — stop when predicate fails
let small = stream.take_while(|x| *x < 10);

// timeout — error if next item takes too long
use std::time::Duration;
let timed = stream.timeout(Duration::from_secs(5));

// chunks — batch items into groups
let batches = stream.chunks(10);
// yields Vec<T> of up to 10 items
```

### Concurrency combinators

```rust
// buffered — run up to N futures concurrently,
// preserving order
let results = url_stream
    .map(|url| fetch(url))
    .buffered(10)
    .collect::<Vec<_>>()
    .await;

// buffer_unordered — same but yields in completion order
let results = url_stream
    .map(|url| fetch(url))
    .buffer_unordered(10)
    .collect::<Vec<_>>()
    .await;
```

### Merging streams

```rust
use tokio_stream::StreamExt;

let merged = stream_a.merge(stream_b);
// Yields items from whichever stream is ready first
```

---

## Creating Streams

### From iterators

```rust
let stream = tokio_stream::iter(vec![1, 2, 3]);
```

### From channels (ReceiverStream)

```rust
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;

let (tx, rx) = mpsc::channel(100);
let stream = ReceiverStream::new(rx);
// Now usable with StreamExt combinators
```

### From async generators (stream::unfold)

```rust
use futures::stream;

let page_stream = stream::unfold(0u32, |page| async move {
    let items = fetch_page(page).await.ok()?;
    if items.is_empty() {
        None  // End of stream
    } else {
        Some((items, page + 1))  // (yield, next_state)
    }
});
```

### From try_unfold for fallible streams

```rust
use futures::stream;

let stream = stream::try_unfold(cursor, |cursor| async move {
    let batch = db.fetch_batch(&cursor).await?;
    if batch.is_empty() {
        Ok(None)
    } else {
        let next_cursor = batch.last().unwrap().id.clone();
        Ok(Some((batch, next_cursor)))
    }
});
```

---

## Pin and Why Streams Require It

Streams are self-referential when they contain `.await`
points. `Pin` guarantees the stream won't move in memory,
which is required for safe polling.

```rust
use std::pin::Pin;
use futures::Stream;

// Returning a concrete stream type — no Pin needed
fn numbers() -> impl Stream<Item = i32> {
    tokio_stream::iter(vec![1, 2, 3])
}

// Returning a dynamic stream — Pin<Box<dyn Stream>> required
fn dynamic_stream(
    use_fast: bool,
) -> Pin<Box<dyn Stream<Item = i32> + Send>> {
    if use_fast {
        Box::pin(tokio_stream::iter(vec![1, 2, 3]))
    } else {
        Box::pin(slow_stream())
    }
}
```

**When you need Pin:**
- `dyn Stream` trait objects (always)
- Storing streams in structs
- Returning different stream types from branches

**When you don't:**
- Using `impl Stream` return types
- Inline stream processing with combinators

---

## async fn in Traits

Since Rust 1.75, `async fn` works directly in trait
definitions:

```rust
trait DataSource {
    async fn fetch(&self, id: &str) -> Result<Data, Error>;
}

impl DataSource for HttpSource {
    async fn fetch(&self, id: &str) -> Result<Data, Error> {
        self.client.get(id).send().await?.json().await
    }
}
```

### When you still need `#[async_trait]`

Native async fn in traits returns an opaque future that
is **not** `Send` by default. If you need `Send` bounds
(common with `tokio::spawn`), you have two options:

```rust
// Option 1: return-position impl Trait (manual Send bound)
trait DataSource {
    fn fetch(
        &self,
        id: &str,
    ) -> impl Future<Output = Result<Data, Error>> + Send;
}

// Option 2: #[async_trait] — still useful for Send
use async_trait::async_trait;

#[async_trait]
trait DataSource: Send + Sync {
    async fn fetch(&self, id: &str) -> Result<Data, Error>;
}
```

**Decision:** Use native `async fn` when the trait is not
used with `tokio::spawn`. Use return-position `impl Trait`
or `#[async_trait]` when you need `Send` futures.

Native `async fn` in traits also does not support dynamic
dispatch (`dyn Trait`). For trait objects, `#[async_trait]`
remains the practical choice.

---

## Practical Patterns

### Paginated API as stream

```rust
use futures::stream::{self, Stream};

fn fetch_all_users(
    client: &Client,
) -> impl Stream<Item = Result<User, Error>> + '_ {
    stream::try_unfold(Some(1u32), move |page| async move {
        let page = match page {
            Some(p) => p,
            None => return Ok(None),
        };
        let resp = client.get_users(page).await?;
        let next = if resp.has_next {
            Some(page + 1)
        } else {
            None
        };
        let items = stream::iter(
            resp.users.into_iter().map(Ok),
        );
        Ok(Some((items, next)))
    })
    .try_flatten()
}
```

### WebSocket message stream

```rust
use tokio_tungstenite::connect_async;
use tokio_stream::StreamExt;

let (ws, _) = connect_async("wss://example.com/ws").await?;
let (write, read) = ws.split();

let mut messages = read
    .filter_map(|msg| msg.ok())
    .filter(|msg| msg.is_text())
    .map(|msg| msg.into_text().unwrap())
    .take_while(|text| text != "CLOSE");

while let Some(text) = messages.next().await {
    handle_message(&text);
}
```

### Database cursor streaming

```rust
fn stream_rows(
    pool: &PgPool,
) -> impl Stream<Item = Result<Row, sqlx::Error>> + '_ {
    sqlx::query("SELECT * FROM large_table")
        .fetch(pool)  // Returns a Stream, not Vec
}

let mut rows = stream_rows(&pool);
while let Some(row) = rows.try_next().await? {
    process(row);
}
```

---

## Anti-Patterns

### Collecting when you only need a few items

```rust
// Bad: loads everything into memory
let all: Vec<_> = stream.collect().await;
let first_ten = &all[..10];

// Good: take before collecting
let first_ten: Vec<_> = stream.take(10).collect().await;
```

### Ignoring backpressure

```rust
// Bad: spawns unbounded tasks from stream
while let Some(item) = stream.next().await {
    tokio::spawn(async move { process(item).await });
}

// Good: use buffered for bounded concurrency
stream
    .map(|item| async move { process(item).await })
    .buffer_unordered(10)
    .for_each(|result| async { handle(result) })
    .await;
```

### Forgetting to pin dynamic streams

```rust
// Won't compile: dyn Stream must be pinned
let stream: Box<dyn Stream<Item = i32>> = ...;
// stream.next() — error: Stream not implemented

// Fix: use Pin<Box<...>>
let stream: Pin<Box<dyn Stream<Item = i32> + Send>> =
    Box::pin(tokio_stream::iter(vec![1, 2, 3]));
```
