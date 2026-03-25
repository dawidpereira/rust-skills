# Async Channels

## mpsc: Work Queue

Multiple producers, single consumer. The workhorse channel for task communication.

```rust
use tokio::sync::mpsc;

let (tx, mut rx) = mpsc::channel::<Command>(100);

// Clone before moving into each task — Sender::clone() is cheap (atomic increment)
for i in 0..4 {
    let tx = tx.clone();
    tokio::spawn(async move {
        let _ = tx.send(Command::Process(i)).await;
    });
}
drop(tx);  // Drop the original so rx.recv() returns None when all tasks finish

while let Some(cmd) = rx.recv().await {
    handle(cmd);
}
```

**Always use `tokio::sync::mpsc`**, not `std::sync::mpsc`
— the std version blocks the runtime thread.

### Request-Response with mpsc + oneshot

```rust
enum Command {
    Get { key: String, reply: oneshot::Sender<Option<String>> },
    Set { key: String, value: String },
}

// Handler task
while let Some(cmd) = rx.recv().await {
    match cmd {
        Command::Get { key, reply } => {
            let value = store.get(&key).cloned();
            let _ = reply.send(value);
        }
        Command::Set { key, value } => {
            store.insert(key, value);
        }
    }
}

// Client
let (reply_tx, reply_rx) = oneshot::channel();
tx.send(Command::Get { key: "foo".into(), reply: reply_tx }).await?;
let value = reply_rx.await?;
```

### Graceful Shutdown

When all senders are dropped, `rx.recv()` returns `None`:

```rust
while let Some(msg) = rx.recv().await {
    process(msg);
}
// All senders dropped — loop exits naturally
```

---

## oneshot: Single Message

Optimized for exactly one message. Common in request-response patterns.

```rust
use tokio::sync::oneshot;

let (tx, rx) = oneshot::channel();

tokio::spawn(async move {
    let result = compute().await;
    let _ = tx.send(result);  // Receiver may have been dropped
});

// With timeout
match tokio::time::timeout(Duration::from_secs(5), rx).await {
    Ok(Ok(result)) => use_result(result),
    Ok(Err(_)) => println!("Sender dropped"),
    Err(_) => println!("Timed out"),
}
```

---

## broadcast: Pub/Sub

All subscribers receive all messages. Messages are cloned to each receiver.

```rust
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<Event>(100);

// Each subscriber gets its own receiver
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

tokio::spawn(async move {
    while let Ok(event) = rx1.recv().await {
        handle_logging(event);
    }
});

tokio::spawn(async move {
    while let Ok(event) = rx2.recv().await {
        handle_metrics(event);
    }
});

tx.send(Event::UserLogin { user_id: 42 })?;
// Both rx1 and rx2 receive this event
```

### Handling Lagged Receivers

If a receiver falls behind, it gets `RecvError::Lagged(n)`:

```rust
loop {
    match rx.recv().await {
        Ok(msg) => process(msg),
        Err(broadcast::error::RecvError::Lagged(n)) => {
            eprintln!("Missed {n} messages, catching up");
        }
        Err(broadcast::error::RecvError::Closed) => break,
    }
}
```

**Note:** Messages must implement `Clone`. Use
`Arc<LargeData>` to avoid expensive clones.

---

## watch: Latest Value

Share the most recent value. Slow receivers skip
intermediate values — they always see the latest.

```rust
use tokio::sync::watch;

let (tx, mut rx) = watch::channel(Config::default());

// Observer: reacts to changes
tokio::spawn(async move {
    while rx.changed().await.is_ok() {
        let config = rx.borrow().clone();
        apply_config(&config);
    }
});

// Updater
tx.send(Config::new_with_port(9090))?;

// Conditional update
tx.send_if_modified(|config| {
    if config.port != new_port {
        config.port = new_port;
        true  // Notify receivers
    } else {
        false  // No change, don't notify
    }
});
```

### watch vs broadcast vs mpsc

|               | watch           | broadcast         | mpsc                    |
| ------------- | --------------- | ----------------- | ----------------------- |
| Receivers get | Latest only     | All messages      | Each message once       |
| Slow receiver | Skips to latest | Gets Lagged error | Sender blocks (bounded) |
| Best for      | Config, state   | Events, logs      | Work queues             |

---

## Backpressure with Bounded Channels

Unbounded channels grow without limit. Bounded channels
apply backpressure — the sender waits (or fails) when
the buffer is full.

```rust
let (tx, rx) = mpsc::channel(100);  // Bounded to 100

// Sender blocks until space available
tx.send(msg).await?;

// Or check without blocking
match tx.try_send(msg) {
    Ok(()) => {},
    Err(TrySendError::Full(msg)) => {
        // Handle backpressure: drop, queue locally, slow down
    }
    Err(TrySendError::Closed(msg)) => {
        // Receiver gone
    }
}

// Or with timeout
tokio::time::timeout(Duration::from_millis(100), tx.send(msg)).await??;
```

**Buffer sizing guidelines:**

- Start with `num_producers * 2` or expected burst size
- Too small → unnecessary blocking
- Too large → defeats backpressure purpose
- Monitor with `tx.capacity()` in production
