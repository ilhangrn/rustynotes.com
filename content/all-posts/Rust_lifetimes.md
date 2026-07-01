+++
title = "Rust lifetimes: tracing references"
date = 2026-07-01
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Lifetimes", "References"]
+++
---
<br>

## (*Eng*) Rust lifetimes: tracing references

Rust moves values on assignment and tracks mutable versus immutable borrows. The final piece is lifetimes: when a function returns a reference, the compiler must prove that reference never outlives the data behind it.

> Lifetimes are compile-time metadata, not runtime code.

### Why lifetimes exist

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

The `'a` annotation tells the compiler that the returned reference lives at least as long as the shorter of the two inputs. Without it, the compiler cannot prove safety.

### The core rule

A reference is valid only while the data it points to exists. If the owner is dropped first, the reference becomes invalid. Rust rejects this at compile time.

| Concept | What happens |
|:--|:--|
| Move | Ownership transfers, old name invalid |
| Borrow | Reference created, owner still responsible |
| Lifetime | Compiler proves reference does not outlive owner |

```rust
let s;
{
    let s1 = String::from("hello");
    s = &s1; // error: s1 dropped before s goes out of scope
}
// println!("{}", s);
```

### elided lifetimes

Many lifetimes can be inferred, so Rust lets you omit them in function signatures. The compiler fills in the blanks using three elision rules.

```rust
fn first_word(s: &str) -> &str {
    // compiler inserts matching lifetime for input and output
}
```

> Not all lifetimes can be elided. Once you return a reference from one of several inputs, you must annotate it explicitly.

### Why strict lifetimes

In C/C++ you can inadvertently return a pointer to a local variable. The program may run for hours before crashing. Rust turns that class of bug into a compile error.

```rust
fn bad() -> &String {
    let s = String::from("oops");
    &s // compile error: `s` does not live long enough
}
```

> Core lesson: lifetimes make Rust's borrow checker complete. Moving and borrowing give you safety for single scopes; lifetimes prove cross-scope references are valid.
