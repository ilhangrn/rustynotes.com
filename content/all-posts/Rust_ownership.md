+++
title = "Rust ownership: the secret sauce"
date = 2026-07-01
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Ownership", "Memory Safety"]
+++
---
<br>

## (*Eng*) Rust ownership: the secret sauce

Rust's headline feature is ownership: every value has exactly one owner, and when that owner goes out of scope, the value is automatically freed. No garbage collector, no `malloc`/`free` bookkeeping — just compile-time guarantees that prevent the two most common C/C++ bugs: use-after-free and double-free.

> Ownership turns memory safety into a type system problem rather than a runtime concern.

### The single-owner rule

```rust
let s1 = String::from("hello");
let s2 = s1;
// s1 is invalid now — ownership moved to s2
// println!("{}", s1); // compile error
```

When you transfer ownership, the compiler invalidates the source. You cannot accidentally use a value after handing it off.

### Borrowing as an alternative

Most of the time you don't need to take ownership. Borrowing lets you use a value without freeing it.

```rust
let s = String::from("hello");
let length = len(&s);
// s is still valid here
```

A reference is a borrowed pointer. It does not outlive the owner.

### The two borrow rules

The borrow checker enforces two rules at compile time:

- You can have any number of immutable borrows at the same time, **or**
- Exactly one mutable borrow

Not both.

```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &s;
// let r3 = &mut s; // error: cannot borrow mutably while immutably borrowed
```

This prevents data races without a runtime cost.

### Why not garbage collection?

Rust could have used a tracing garbage collector. Instead it chose ownership because:

- No background collector thread pauses
- No mark-and-sweep overhead
- Memory layout is predictable for embedded and systems code
- Bugs that GC hides become compile errors in Rust

> Core lesson: ownership and borrowing let Rust match C/C++ performance while eliminating whole classes of memory bugs.
