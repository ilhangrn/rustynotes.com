+++
title = "Rust error handling: Option and Result"
date = 2026-07-01
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Error Handling", "Option", "Result"]
+++
---
<br>

## (*Eng*) Rust error handling: Option and Result

Rust turns many runtime errors into compile-time guarantees through two enums: `Option<T>` and `Result<T, E>`. Unlike null-pointer exceptions, you cannot ignore these values. The compiler forces you to handle them before using the underlying data.

> Missing-value and failure cases become explicit types, not hidden assumptions.

### Option<T>: a value that may not exist

```rust
enum Option<T> {
    Some(T),
    None,
}
```

Use `Option` when a value is optional.

```rust
let text = map.get("missing");
match text {
    Some(t) => println!("{}", t),
    None => println!("not found"),
}
```

Shorthand forms:

```rust
if let Some(t) = text {
    println!("{}", t);
}

let t = text.unwrap_or("default");
```

### Result<T, E>: an operation that may fail

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Use `Result` for operations that can fail with a typed reason.

```rust
match fs::read_to_string(path) {
    Ok(data) => process(data),
    Err(Error::NotFound) => eprintln!("missing file"),
    Err(Error::PermissionDenied) => eprintln!("access denied"),
    Err(e) => eprintln!("io failed: {}", e),
}
```

### Pattern matching as try/catch

The closest equivalent to try/catch with per-error behavior is `match` on `Result`.

| Pattern | Purpose |
|:--|:--|
| `match result { ... }` | exhaustive per-error handling |
| `if let Err(e) = result { ... }` | handle one variant |
| `result.map_err(|e| handle(e))?` | transform then propagate |
| `?` operator | return error to caller immediately |

```rust
match parse_config(path) {
    Ok(cfg) => run(cfg),
    Err(ConfigError::Missing(key)) => eprintln!("missing key: {}", key),
    Err(ConfigError::Parse(e)) => eprintln!("parse failed: {}", e),
    Err(e) => return Err(e),
}
```

### The ? operator

`?` unwraps `Ok` and returns `Err` early from the current function. It replaces nested `match` for linear happy-path flow.

```rust
fn load() -> Result<Config, ConfigError> {
    let text = fs::read_to_string(path)?;
    let cfg = parse(&text)?;
    Ok(cfg)
}
```

### Why not exceptions

Exceptions hide control flow. A function can return normally or throw from anywhere, making reasoning about state harder. Rust errors are values. They flow through the type system, are explicit in signatures, and cannot be silently ignored.

> Core lesson: handle missing data with `Option`, handle recoverable failure with `Result`, and reserve panics for unrecoverable program invariants.
