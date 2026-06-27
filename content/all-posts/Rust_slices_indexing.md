+++
title = "Rust slices and indexing"
date = 2026-06-26
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Learning", "Slices", "Indexing"]
+++
---
<br>

## (*Eng*) Rust slices and indexing

Three questions about the same underlying rule — sized vs unsized types.

### How to set an array of zeros

```rust
let zeros: [i32; 5] = [0; 5];
```

The `[val; N]` repeat expression evaluates `val` once and copies it `N` times. Works for any `Copy` type.

```rust
let zeros: Vec<i32> = vec![0; 5];   // heap version
let zeros: [f64; 10] = [0.0; 10];  // or float
```

**Embedded use case.** Initialise a register buffer in one line, no memset:

```rust
let adc_buffer: [u16; 256] = [0; 256];
```

**The gotcha.** The length is part of the type — it must be a compile-time constant.

```rust
let n = 5;
// let bad: [i32; n] = [0; n];  // ❌ n is not a const
let good: Vec<i32> = vec![0; n];  // ✅ Vec is runtime-sized
```

### Why `&a[1..4]` instead of `a[1..4]`

You cannot bind a range index by value:

```rust
let a = [10, 20, 30, 40, 50];
let slice = a[1..4];  // ❌ expected &[i32], found [i32]
```

The `[]` operator desugars into a method call:

```rust
// a[1..4] means *a.index(1..4)
// Index::index(&self, idx) -> &Self::Output
// So * dereferences &[i32] back to [i32]
// And [i32] is unsized — the compiler does not know its length at compile time
```

Unsized types cannot live on the stack. The fix is to borrow the result back into a fat pointer:

```rust
let slice = &a[1..4];  // ✅ &[i32] — two words: data pointer + length
```

`&[i32]` has a known size (16 bytes on 64-bit), so the stack can hold it.

**C analogy.** You don't copy array slices by value in C either:

```c
int arr[5] = {10, 20, 30, 40, 50};
int *slice = &arr[1];  // pointer into the original
```

Rust's `&[i32]` is that same pointer, but with the length carried alongside — bounds-checked and no off-by-ones.

**Panic-safe alternative.**

```rust
let slice = a.get(1..4);  // Option<&[i32]>
```

Returns `None` instead of panicking when the range is out of bounds.

### Why `size_of_val` can trick you

```rust
let a: [u8; 4] = [10, 20, 30, 40];

size_of_val(&a)       // 4 — size of the [u8; 4] value
size_of_val(&a[1..3]) // 2 — size of the [u8] slice (2 elements)
```

`size_of_val` looks through the reference and measures the **pointee**, not the pointer.

```rust
size_of::<&[u8; 4]>()  // 8  — thin pointer (one word)
size_of::<&[u8]>()     // 16 — fat pointer (data ptr + length)
```

`size_of` on the reference type measures the **pointer itself**.

| Expression | measures | size |
|---|---|---|
| `size_of_val(&a)` | the array `[u8; 4]` | 4 |
| `size_of_val(&a[1..3])` | the slice `[u8]` of 2 elements | 2 |
| `size_of::<&[u8; 4]>()` | the thin pointer | 8 |
| `size_of::<&[u8]>()` | the fat pointer | 16 |

This is the same reason `&a[1..4]` is required — `[u8]` is unsized, but `&[u8]` is a known-size fat pointer the compiler can place on the stack.

### The big picture

| Problem | C habit | Rust way | Why |
|---|---|---|---|
| Zero-initialised array | `int buf[256] = {0}` | `let buf: [u16; 256] = [0; 256]` | Repeat expression, no memset |
| Slice of existing array | `int *p = &arr[1]` | `let s = &a[1..4]` | `[]` returns unsized `[T]`, `&` makes it `&[T]` |
| Check slice size | `sizeof` on array | `size_of_val(&s)` | Measures pointee, not pointer |

**The core lesson.** Rust gives you the same low-level control as C — zero-cost slices, no hidden copies — but forces you to be explicit about sized vs unsized types. When the compiler says `[i32]` is unsized, it is telling you "you need a reference, not a bare slice value." Once that clicks, `&a[1..4]` becomes instinct.
