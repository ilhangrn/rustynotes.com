+++
title = "Rust ? in main and the Result return trick"
date = 2026-07-17
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "error handling", "main"]
+++
---
<br>

## (*Eng*) Rust ? in main and the Result return trick

`?` is one of Rust's most useful operators, but it is often treated as magic. It is not a normal expression. It is shorthand for returning early from a function when something fails. That difference matters most inside `main`, because `main` has no error slot by default.

> `?` asks: "If this fails, what value should I return?" The compiler needs an answer.

### What `?` really does

Consider this parsing example from rustlings `errors3.rs`:

```rust
fn total_cost(item_quantity: &str) -> Result<i32, ParseIntError> {
    let qty = item_quantity.parse::<i32>()?;
    Ok(qty * cost_per_item + processing_fee)
}
```

If `parse()` returns `Ok(v)`, `?` unwraps it and `qty` becomes `v`. If it returns `Err(e)`, `?` returns `Err(e)` immediately from `total_cost`.

It is equivalent to:

```rust
let qty = match item_quantity.parse::<i32>() {
    Ok(v) => v,
    Err(e) => return Err(e),
};
```

Because `total_cost` already promises `Result<i32, ParseIntError>`, the compiler knows exactly which type to return when things go wrong.

### The `main` surprise

Now try the same node inside `main`:

```rust
fn main() {
    let cost = total_cost(pretend_user_input)?;
}
```

This fails to compile. `main` implicitly returns `()`. There is no error payload available. When `?` tries to return an error, the compiler asks "return it as what?" and gets no answer.

That is why `main` needs a custom signature:

```rust
fn main() -> Result<(), ParseIntError> {
    let cost = total_cost(pretend_user_input)?;
    Ok(())
}
```

Now `main` can legally return an error. `?` propagates `ParseIntError` out of `total_cost`, becomes the return value of `main`, and the runtime prints it automatically before exiting non-zero. This is the canonical rustlings fix in `errors3.rs`.

### `?` with non-Rust error types

Rust can convert some foreign error types automatically if they implement `From`. The idiomatic closure of `main` is:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cost = total_cost(pretend_user_input)?;
    Ok(())
}
```

`Box<dyn std::error::Error>` is a trait object that can hold almost any error type. Each `?` on a different error branch converts automatically via `Into<Box<dyn std::error::Error>>`.

### An embedded analogy

Think of functions and call stacks. In embedded C, if a helper wants to report failure it either returns an `int` status or sets an out-parameter. `main` that calls `printf` is equivalent: without a status return, there is nowhere to put a failure code. Adding `Result<(), E>` is exactly like changing `void main(void)` into `int main(void)` so failure can propagate up the call stack.

The difference is that C systems often ignore `main`'s return value. In Rust, `Result<(), E>` from `main` is special: the runtime prints the error and exits non-zero. You get a stack-trace-like failure signal for free.

### Where `?` does not work

* On values that are not `Result` or `Option`.*
* Inside functions that return plain concrete types without `try` blocks.
* Inside closures whose inferred closure traits do not permit returning the error.

### Practical rules

| Situation | Fix |
|:--|:--|
| `?` inside `main` | Add `-> Result<(), E>` |
| Mixed error types | Use `Box<dyn std::error::Error>` |
| Want to handle instead of propagate | Use `match` or `if let` |
| Need custom fallback | Use `unwrap_or(default)` |

If `?` fails to compile, read the first message: the compiler is almost certainly asking you to "make this function return a `Result`". Give the function a return type, not a panic.

### Reference checklist

- `?` never panics. It returns early.
- It works only inside functions returning `Result` or `Option`.
- `main` defaults to `()`, so `?` fails there unless you change `main`'s return type.
- `Result<(), Box<dyn std::error::Error>>` is the catch-all `main` signature in real projects.
