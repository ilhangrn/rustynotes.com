+++
title = "Rust borrow checker"
date = 2026-07-01
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Ownership", "Borrowing"]
+++
---
<br>

## (*Eng*) Rust borrow checker

Rust's most famous feature is the borrow checker. It enforces memory safety without a garbage collector and without manual `malloc` and `free`. The rule is simple at the surface: every value has exactly one owner, and when that owner goes out of scope, the value is dropped.

> Ownership gives you safety by default. No leaks, no double-frees, no use-after-free.

### Ownership in practice

```rust
let s1 = String::from("hello");
let s2 = s1;
// s1 is no longer valid — ownership moved to s2
// println!("{}", s1); // compile error
```

When you assign a `String` to another variable, ownership moves. The first variable is invalidated automatically. This is a compile-time guarantee, not a runtime check.

### Borrowing

If you need to use a value without taking ownership, you borrow it.

```rust
let s = String::from("hello");
let len = calculate_length(&s);
// s is still valid here
```

A borrow is a reference. The borrower does not drop the value when it goes out of scope. The owner is still responsible for cleanup.

### The three closure traits

Closures capture variables from their environment. Rust classifies them by how they capture.

| Trait | Capture style | Call count |
|:--|:--|:--|
| `FnOnce` | Takes ownership | Once |
| `FnMut` | Mutable borrow | Multiple times |
| `Fn` | Immutable borrow | Multiple times |

```rust
let mut nums = vec![1, 2, 3];
let mut consume = |x| nums.push(x);
consume(4);
// nums is now [1, 2, 3, 4]
```

> You can call a closure by reference, mutable reference, or by value. The trait it implements tells you which it accepts.

### The mutable borrow rule

You can have either one mutable borrow or any number of immutable borrows. Not both at the same time.

```rust
let mut s = String::from("hello");
let r1 = &s;        // immutable borrow
let r2 = &s;        // immutable borrow
// let r3 = &mut s; // compile error: cannot borrow as mutable
```

This prevents data races at compile time. If one part of the code is reading through an immutable reference, no other code can mutate the value until all references are gone.

### Why it matters

The borrow checker is what replaces garbage collection in Rust. It decides when memory is freed by tracking references at compile time. The rules feel strict at first, but they eliminate whole classes of bugs before your program runs.

> Core lesson: ownership and borrowing are not extra work. They are the mechanism that makes Rust fast and safe without a runtime.
