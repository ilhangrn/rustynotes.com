+++
title = "Rust parsing, unwrap() panic and typed error propagation"
date = 2026-07-17
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "error handling", "parse", "library"]
+++
---
<br>

## (*Eng*) Rust parsing, unwrap() panic and typed error propagation

Parsing user input is one of the most common failure sources in Rust programs. The standard library gives you `str::parse()`, which returns `Result<T, ParseIntError>`. That return type is the clue: parsing can fail, so you must handle the failure explicitly.

When beginners reach rustlings `errors6.rs`, the instinct is often to call `.unwrap()` on `parse()` because it works in tutorials. The exercise exists to prove why that instinct is wrong in library code, and how to propagate the error through a typed enum instead.

> `parse()` does not crash. `unwrap()` does. The failure is not the problem; ignoring it is.

### The bug in `errors6.rs`

Look at the method under test:

```rust
fn parse(s: &str) -> Result<Self, ParsePosNonzeroError> {
    let x: i64 = s.parse().unwrap(); // BAD
    Self::new(x).map_err(ParsePosNonzeroError::Creation)
}
```

`s.parse()` returns `Result<i64, ParseIntError>`. On valid input like `"42"` it returns `Ok(42)`. On invalid input like `"not a number"` it returns `Err(ParseIntError)`. Calling `.unwrap()` on `Err` **panics the program**.

The test on line 51 makes the intended behavior explicit:

```rust
assert!(matches!(
    PositiveNonzeroInteger::parse("not a number"),
    Err(ParsePosNonzeroError::ParseInt(_)),
));
```

It expects bad input to become a normal `Err`, not a crash. Once you see the test, the rule is clear: library code should propagate failures, not panic on them.

### The fix: `?` with `map_err`

`s.parse()` returns `ParseIntError`, but `parse()`'s signature promises `ParsePosNonzeroError`. You need a bridge between those two types:

```rust
fn parse(s: &str) -> Result<Self, ParsePosNonzeroError> {
    let x: i64 = s.parse().map_err(ParsePosNonzeroError::ParseInt)?;
    Self::new(x).map_err(ParsePosNonzeroError::Creation)
}
```

The expression reads left to right:

1. `s.parse()` gives `Result<i64, ParseIntError>`.
2. `map_err(ParsePosNonzeroError::ParseInt)` wraps the error into the custom enum variant.
3. `?` propagates the wrapped error immediately if parsing fails.

If `s` is `"42"`, `?` unwraps `42` and `parse()` continues to `Self::new(42)`. If `s` is `"abc"`, `?` returns `Err(ParsePosNonzeroError::ParseInt(_))` from `parse()`.

### Why not `Box<dyn Error>` here?

`Box<dyn Error>` is the catch-all pattern you saw in `errors5.rs` `main`. It is acceptable there because `main` is at the top of the application and callers only need to print the error. For a **library method** like `PositiveNonzeroInteger::parse`, a catch-all is the wrong choice.

Callers need to know *why* parsing failed. With `Box<dyn Error>`, the only tool is `println!`. With a typed enum, callers can match on variants:

```rust
match PositiveNonzeroInteger::parse(input) {
    Ok(n) => use(n),
    Err(ParsePosNonzeroError::ParseInt(_)) => retry_or_log_parse_error(),
    Err(ParsePosNonzeroError::Creation(_)) => retry_or_log_range_error(),
}
```

Structured errors are the difference between "it failed" and "it failed because of X."

### The enum as an error contract

Notice how the enum is just two variants:

```rust
enum ParsePosNonzeroError {
    ParseInt(ParseIntError),
    Creation(CreationError),
}
```

That captures exactly the two failure modes:

- The input was not a number.
- The number was outside the allowed range.

A caller does not need to read documentation to know what can go wrong. The type tells them.

In `errors5.rs` you saw `Box<dyn Error>` used to merge unrelated errors. Here you see the opposite: an enum used to keep them separate. Both are correct; they serve different audiences.

### Embedded analogy

Think of reading a configuration string from flash or a UART. If you call `atoi()` and then validate the range, you have two failure modes: malformed input and out-of-range value. A C API often papers over that with a single `int status`. In Rust, a `ParsePosNonzeroError` enum is like having two status bits you can test individually, without casting or bit-twiddling.

### Practical rules

| Situation | Error style |
|:--|:--|
| Top-level `main` in an app | `Box<dyn Error>` |
| Library API for external callers | Typed `enum Error` |
| One-off script | `Box<dyn Error>` or `Result<(), E>` |
| Propagation inside your crate | Typed `enum Error` |
| Wrapping another library's error | `map_err(MyError::Other)` |

Whenever you write a function that returns `Result`, ask: "does the caller need to react differently to different failures?" If yes, build an enum. If no, a catch-all may be acceptable.

### Common mistakes

**Using `unwrap()` in library APIs**
`unwrap()` is for known-safe cases inside your own crate. It should not appear in public methods where caller input reaches it.

**Wrapping errors without adding value**
Every `map_err` should preserve information. If you wrap a `ParseIntError` but discard its content, debugging becomes harder. Prefer `enum Error { ParseInt(ParseIntError), ... }`.

**Confusing `Box<dyn Error>` with library design**
`Box<dyn Error>` is an application-level convenience. Library code contract should be explicit.

### Reference checklist

- `str::parse()` returns `Result<T, ParseIntError>`; always handle the `Err`.
- `.unwrap()` on `parse()` is only safe when the input is truly guaranteed.
- Use `?` + `map_err(YourEnum::Variant)` to convert into a custom enum.
- Use typed enums in library code; use `Box<dyn Error>` in binary `main`.
- The test drives the shape of your error enum.
