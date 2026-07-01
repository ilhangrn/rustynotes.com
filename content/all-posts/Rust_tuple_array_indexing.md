+++
title = "Rust tuple and array indexing"
date = 2026-07-01
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Learning", "Indexing", "Tuples"]
+++
---
<br>

## (*Eng*) Rust tuple and array indexing

They both look like indexing, but one is a compile-time field name and the other is a runtime offset. Mixing them up produces a confusing error.

```rust
let arr = [10, 20, 30];
let tup = (10, "hello", 3.14);

arr[1]    // ✅ 20
tup.1     // ✅ "hello"
tup[1]    // ❌ error[E0608]: cannot index into a tuple
arr.1     // ❌ error[E0610]: `[{integer}; 3]` is not a tuple
```

> Tuple access is a field lookup. Array access is an `Index` trait call. They are different operations that happen to both retrieve an element by position.

### What each does under the hood

| Feature                     | Array `arr[i]`                             | Tuple `tup.0`                                      |
| :-------------------------- | :----------------------------------------- | :------------------------------------------------- |
| **Resolution**              | Runtime (via `Index` trait)                | Compile time (field access)                        |
| **Index expression**        | Any `usize` value — variable, loop counter  | A literal integer — must be known to the compiler  |
| **Type of each element**    | All the same type `T`                      | Each field can be a different type                 |
| **Bounds check**            | Runtime — panics on out-of-range            | Compiler rejects non-existent fields               |
| **Dynamic index**           | `arr[i]` where `i` is a variable — fine    | `tup.i` where `i` is a variable — impossible       |

### The embedded C analogy

In C, there is no tuple type. You reach for a `struct`:

```c
struct sensor {
    uint32_t raw;
    const char *name;
    float scale;
};

struct sensor s = { 1024, "ADC0", 3.3 };
s.raw;   // field access — known at compile time
```

And for homogeneous data, you use an array:

```c
uint16_t buf[8];
buf[i];  // runtime offset — can be any index
```

Rust tuples are anonymous structs with numeric field names. The `.0`, `.1`, `.2` are those field names, not indices. The compiler stamps out exactly one instruction — a fixed offset into the struct layout — and moves on.

The `[]` operator on an array is an actual method call. It adds the index to the base pointer, multiplies by `size_of::<T>()`, and dereferences. All of that happens at runtime.

### Why Rust chose this design

**Tuples are heterogeneous.** Each field has its own type. To use `tup[i]` with a variable `i`, the compiler would need to know at compile time what type the expression `tup[i]` returns. It can't — the type depends on the value of `i`.

```rust
let tup = (42_u32, "hello");
// If tup[i] were allowed, what type is it?
// i == 0 -> u32
// i == 1 -> &str
// The compiler would need a sum type at every tuple access
```

**Arrays are homogeneous.** Every element is `T`. `arr[i]` always returns `T` regardless of `i`. No type ambiguity — the compiler can defer everything to runtime.

> Tuples are for structures where each position has meaning. Arrays are for collections where every element is interchangeable.

### Dynamic dispatch with tuples — the workaround

If you genuinely need to index into a tuple by a runtime value, you pattern-match:

```rust
fn get_nth(tup: (u32, &str, f64), i: usize) -> ?? {
    match i {
        0 => tup.0,
        1 => tup.1,
        2 => tup.2,
        _ => panic!("out of range"),
    }
}
```

But notice the return type — each arm returns a different type, so the function must return an enum or use dynamic dispatch. This tells you Rust's tuple was never designed for dynamic access.

### Summary

| Concept                         | Array `[T; N]`               | Tuple `(A, B, C, ...)`                     |
| :------------------------------ | :--------------------------- | :----------------------------------------- |
| Syntax                          | `arr[i]`                     | `tup.0`                                    |
| Index type                      | `usize` at runtime           | Integer literal at compile time            |
| Trait                           | `Index<usize>`               | Named fields (no trait involved)           |
| Heterogeneous                   | No                           | Yes                                        |
| Dynamic access                  | Yes                          | No — use match or destructure              |

**The core lesson.** When you write `tup[0]` and get a type error, the fix is not to search for a trait to implement — it is to recognise that `tup.0` is a field access, not an index. Once you see tuples as anonymous structs rather than small arrays, the whole thing clicks.
