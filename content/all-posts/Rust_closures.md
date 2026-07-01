+++
title = "Rust closures: capture and call"
date = 2026-07-01
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Closures", "Functions"]
+++
---
<br>

## (*Eng*) Rust closures: capture and call

Rust's closures are anonymous functions you can store in variables or pass around. Their defining property is capture: they can take values from the surrounding scope.

> Closures are lightweight function literals with access to their environment.

### The three capture traits

| Trait | Capture style | Call count | Mutation allowed |
|:--|:--|:--|:--|
| `FnOnce` | Takes ownership | Once | No |
| `FnMut` | Mutable borrow | Multiple times | Yes |
| `Fn` | Immutable borrow | Multiple times | No |

### FnOnce: consume the environment

```rust
let nums = vec![1, 2, 3];
let consume = |x| nums.push(x);
consume(4);
// nums is now [1, 2, 3, 4]
// This closure takes ownership of `nums`
```

A `FnOnce` closure invalidates captured values when called. You can call it exactly once.

### FnMut: mutate through a borrow

```rust
let mut list = vec![1, 2, 3];
let mut adder = |x| list.push(x);
adder(4);
adder(5);
// list is now [1, 2, 3, 4, 5]
```

This closure borrows `list` mutably. It can be called repeatedly because the borrow ends after each call.

### Fn: read without changing

```rust
let list = vec![1, 2, 3];
let printer = |x| println!("{}", x);
printer(2);
```

An `Fn` closure only reads captured values through immutable borrows. Multiple calls are allowed, and any number of `Fn` closures can exist together.

### Moving values in

Values captured from the environment can be moved into the closure or borrowed from it.

```rust
let name = String::from("rust");
let greet = || println!("hello {}", name);
// `name` is immutably borrowed
```

If the closure needs to store the value, it moves ownership instead:

```rust
let name = String::from("rust");
let greet = move || println!("hello {}", name);
// `name` moved into the closure
```

### Passing closures to functions

Functions accept closures with trait bounds just like any generic type.

```rust
fn apply<F>(f: F)
where
    F: FnOnce(),
{
    f();
}
```

```rust
let x = 5;
apply(|| println!("value is {}", x)); // captures by reference
```

> Core lesson: closures in Rust are not just anonymous functions. Their capture mode is part of the type system, and the compiler selects the right trait based on how the closure uses captured values.
