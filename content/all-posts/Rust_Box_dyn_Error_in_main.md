+++
title = "Rust Box<dyn Error> as a catch-all error in main"
date = 2026-07-17
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "error handling", "Box", "trait object"]
+++
---
<br>

## (*Eng*) Rust Box<dyn Error> as a catch-all error in main

The pattern `-> Result<(), Box<dyn Error>>` appears frequently in `main`, but it is often introduced with little explanation. It is not magic. It is a deliberate escape hatch for one specific situation: when different parts of your code can fail with different error types, and you still want to use `?` inside `main`.

This is exactly what rustlings `errors5.rs` demonstrates. Once you understand why `Box` appears, the syntax becomes mechanical.

> When multiple error types meet, `Box<dyn Error>` is the type-level bridge that lets them all flow through the same `?`.

### The problem: `?` needs one error type

Recall the rule: `?` works only inside a function that promises a single error return type. If a function calls both `parse()` and `PositiveNonzeroInteger::new(x)`, it is dealing with two incompatible error types:

```rust
let x: i64 = pretend_user_input.parse()?;
// parse() returns Result<i64, ParseIntError>

PositiveNonzeroInteger::new(x)?
// new() returns Result<POS, CreationError>
```

The first `?` can yield `ParseIntError`. The second can yield `CreationError`. `main` can declare only one return type. Without a shared type, these errors cannot coexist.

### Why plain `dyn Error` is not enough

`dyn Error` is a trait object. It describes behavior, not a sized value. Rust needs to know how much space the error occupies because return values live on the stack by default. A trait object itself has no fixed size — it could be any concrete type that implements `Error`.

So a bare `Result<(), dyn Error>` is invalid syntax because the compiler cannot allocate stack space for an unknown-sized return value. You need a pointer wrapper to hold the trait object. `Box<dyn Error>` is the most common one.

### What `Box` does here

`Box<T>` moves a value onto the heap and returns a heap pointer of fixed width. `Box<dyn Error>` therefore has a known size regardless of the underlying concrete error. Both `ParseIntError` and `CreationError` can be boxed into the same return slot because `Box` erases the size difference.

The flow is:

```rust
fn main() -> Result<(), Box<dyn Error>> {
    let x: i64 = pretend_user_input.parse()?;
    //       ^  if Err(ParseIntError), it is converted into Box<dyn Error>
    println!("output={:?}", PositiveNonzeroInteger::new(x)?);
    //                                              ^^^ if Err(CreationError), it is also converted
    Ok(())
}
```

Each time `?` returns an error, Rust uses `From` to convert the concrete error into `Box<dyn Error>`. The caller sees the same trait-object type every time.

### How `From` makes it seamless

The conversion from specific errors to `Box<dyn Error>` depends on `From`. Rust provides an automatic `From<E>` implementation for `Box<dyn Error>` whenever `E: Error + 'static`. That is why you do not need to write `.map_err(|e| Box::new(e) as Box<dyn Error>)` everywhere. The `?` operator does the boxing for you.

In `errors5.rs` both `ParseIntError` and `CreationError` implement `std::error::Error`:

- `ParseIntError` has a built-in `Error` impl.
- `CreationError` gets its `Error` impl manually on line 34.

Because both satisfy the trait bound, both can flow through the same `?` into the same `Box<dyn Error>`.

### Embedded analogy

Consider an embedded application with multiple peripheral driver errors. If `flash_read()` returns `FlashError` and `uart_write()` returns `UartError`, a single generic error handler cannot distinguish them without a shared supertype or union type. In C, you often see an `int` status, or a callback with a `void* ctx` that you cast back. `Box<dyn Error>` is Rust's safer equivalent: a dynamic interface behind a pointer, with automatic conversion instead of manual casting.

### When to use this pattern

- In `main` during initial exploration.
- In short CLI tools or scripts where caller code does not care which subsystem failed.
- When there are only a few `?` calls and you do not want to define a unified `enum Error`.

When to avoid it:

- In library code. Callers should be able to match on concrete error variants. Prefer a real `enum Error` so users can react to `Creation::Negative` versus `ParseInt`.
- When you need structured error handling, logging, or backtraces. A trait object hides all that.

### Companion patterns

| Pattern | Use when |
|:--|:--|
| `Result<T, E>` | One crisp error type in libraries |
| `anyhow::Error` | Ergonomic app-level catch-all |
| `Box<dyn Error>` | Standard library catch-all |
| `enum AppError { A(AErr), B(BErr) }` | Structured cross-subsystem errors |

### Reference checklist

- `dyn Error` has no size — it needs a pointer.
- `Box<dyn Error>` is the standard heap-based catch-all.
- `?` relies on `From` to convert concrete errors into the function's declared error type.
- This pattern is acceptable in binary `main`.
- Prefer concrete error enums in library code.
