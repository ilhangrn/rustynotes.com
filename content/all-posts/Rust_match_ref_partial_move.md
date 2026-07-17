+++
title = "Rust move, partial move and ref in match"
date = 2026-07-17
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Ownership", "Pattern Matching"]
+++
---
<br>

## (*Eng*) Rust move, partial move and ref in match

Pattern matching is not just a cleaner `switch`. In Rust, `match` participates in ownership transfer. That means a pattern like `Message::Move { x, y }` can silently move the inner value, move only a subset, or borrow the whole thing depending on whether the matched value is owned.

These are not corner cases. Getting this wrong is exactly why many beginners hit `use of partially moved value` after a `match`, and why rustlings exercises such as `options3.rs` introduce `ref p`.

> In Rust, the pattern itself decides whether data is moved, partially moved, or borrowed.

### Move in a match

When `match` binds a value by pattern, the matched arm takes ownership of the bound parts.

```rust
enum Message {
    Move { x: i32, y: i32 },
    Quit,
}

let msg = Message::Move { x: 10, y: 20 };

match msg {
    Message::Move { x, y } => println!("x={x}, y={y}"),
    Message::Quit => println!("quit"),
}

// msg is partially moved here
// println!("{:?}", msg); // compile error
```

`x` and `y` are plain integers, which implement `Copy`, so moving them does not invalidate `msg` for later use. But if the fields were non-`Copy` types, such as `String`, you would not be allowed to use `msg` after the arm.

### Partial move

A partial move happens when you move only some fields of a value. The moved fields become inaccessible on the original value. The untouched fields stay reachable.

```rust
struct Payload {
    id: u32,
    data: Vec<u8>,
}

let p = Payload { id: 1, data: vec![1, 2, 3] };

match p {
    Payload { id, .. } => println!("id={id}"),
}

// `p.data` was not moved, but `p.id` was copied
// println!("{:?}", p.data);     // still works
// println!("{:?}", p);          // compile error: partial move
```

The struct pattern `Payload { id, .. }` moves only `id`. Since `id` is `Copy`, `p` remains usable. `p.data` was never touched by the pattern, so that field is still usable as well. The compiler only forbids access to the entire partially-moved value when extra move-sensitive fields exist.

The error `use of partially moved value` is the moment this becomes unmistakable. Another point: `if let` and `while let` follow the same pattern rules as `match`. They do not get an exemption from ownership.

### What `ref` fixes

`ref` is the bridge between borrowing and pattern matching. By default, a pattern like `Some(p)` moves the inner value out of the `Option`. Adding `ref` turns the binding into a borrow instead.

```rust
let maybe = Some(vec![1, 2, 3]);

match maybe {
    Some(ref nums) => println!("len={}", nums.len()),
    None => println!("empty"),
}

println!("still owned: {:?}", maybe);
```

Without `ref`, `maybe` would be partially moved after the `Some` arm. With `ref`, `nums` is `&Vec<i8>`, and `maybe` stays fully alive.

### `ref mut` for mutable borrows

The same logic applies to mutable borrows. `ref mut` is the pattern-level equivalent of `&mut`.

```rust
let mut maybe = Some(vec![1, 2, 3]);

match maybe {
    Some(ref mut nums) => nums.push(4),
    None => {}
}

println!("modified: {:?}", maybe);
```

When you need to mutate a borrowed value inside a pattern, `ref mut` hands you `&mut T`. Without it, you would be trying to move out of borrowed content, which Rust prevents.

### One analogy from embedded C

In embedded C, passing a struct by value to a function copies the whole structure unless you pass a pointer. Borrowing is how you avoid the copy cost while keeping the original intact.

In Rust, plain pattern bindings such as `Point { x, y }` behave more like passing a struct by value. `ref` behaves like passing `&p`: the callee borrows the caller's data. `ref mut` behaves like passing `p` as a non-const pointer: borrow but allow mutation.

### Practical rules

| Pattern | Meaning |
|:--|:--|
| `Some(x)` | Move `x` out of `Some` |
| `Some(ref x)` | Borrow `x` as `&T` |
| `Some(ref mut x)` | Mutable borrow `x` as `&mut T` |
| `Point { a, b }` | Move `a` and `b` if non-`Copy` |
| `Point { ref a, ref b }` | Borrow both fields |

If you need to use the original value after a `match`, reach for `ref` first. If you need to mutate it, reach for `ref mut`. If you genuinely intend to consume it, move without `ref`.

> Borrowing in patterns is not a special case. It is the same ownership model you already know, extended to `match`, `if let`, and `while let`.

### Common mistakes

**"Use of partially moved value" after match**
You bound a non-`Copy` field inside a pattern. Either borrow with `ref` or accept that the original value is consumed.

**Forgetting `ref` on non-`Copy` values**
This is the exact rustlings-style error. `Some(vec)` moves a `Vec`. `Some(ref vec)` borrows it.

**Using `ref` on `Copy` types**
You can, but it adds indirection for no benefit. The compiler auto-copies `u32`, `i32`, and other scalar types. Reserve `ref` for non-`Copy` types where a move would invalidate the source.

### Reference checklist

- `match`, `if let`, and `while let` all participate in ownership transfer.
- A pattern with plain identifiers moves non-`Copy` data by default.
- Partial move leaves some fields usable and others consumed.
- `ref` borrows in pattern position.
- `ref mut` mutably borrows in pattern position.
- Use `ref` whenever the original value must remain valid after the arm.
