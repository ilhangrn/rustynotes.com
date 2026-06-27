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

Three questions that trip up the same concept — why `[0; 5]`, why `&a[1..4]`, and what `size_of_val` actually measures.

### How to set an array of zeros

```rust
let zeros: [i32; 5] = [0; 5];
```

The `[val; N]` repeat expression evaluates `val` once and copies it `N` times. Works for any `Copy` type.

```rust
let zeros: Vec<i32> = vec![0; 5];  // heap version
let zeros: [f64; 10] = [0.0; 10]; // floats work the same
```

**Why this matters in embedded.** Initialising a large register-buffer array:

```rust
let adc_buffer: [u16; 256] = [0; 256];  // all channels zeroed
```

One line, zero loops, no memset.

**The gotcha.** The length is part of the type — must be a compile-time constant.

```rust
let n = 5;
// let bad: [i32; n] = [0; n];  // ❌ n is not a const
let good: Vec<i32> = vec![0; n];  // ✅ Vec is runtime-sized
```

### Why `&a[1..4]` instead of `a[1..4]`

You cannot write:

```rust
let a = [10, 20, 30, 40, 50];
let slice = a[1..4];  // ❌ expected &[i32], found [i32]
```

The `[]` operator desugars. Here is what the compiler actually does:

```rust
// a[1..4] is syntactic sugar for:
*a.index(1..4)

// Index::index returns &Self::Output
// So * dereferences &[T] back to [T]
// And [T] is unsized — no compile-time known length
```

You cannot place an unsized value on the stack. The fix is to borrow it back:

```rust
let slice = &a[1..4];  // ✅ &[i32] — a fat pointer (ptr + len)
```

`&[i32]` is a known-size type (two words: data pointer + length), so the stack can hold it.

**C analogy.** You don't copy array slices by value either:

```c
int arr[5] = {10, 20, 30, 40, 50};
// int slice[3] = &arr[1];  // no, can't return array from function
int *slice = &arr[1];       // pointer into the original
```

Rust's `&[i32]` is that same pointer, but with the length carried alongside — bounds-checked, no off-by-one.

**Safe alternative.**

```rust
let slice = a.get(1..4);  // Option<&[i32]>
```

Same type, wrapped in `Option` instead of panicking on out-of-range.

### Why `size_of_val` surprises you

Two different questions:

```rust
let a: [u8; 4] = [10, 20, 30, 40];

size_of_val(&a)       // 4 — size of the [u8; 4] value
size_of_val(&a[1..3]) // 2 — size of the [u8] slice (2 elements)
```

`size_of_val` looks **through** the reference and measures the pointee.

```rust
size_of::<&[u8; 4]>()  // 8  — thin pointer (one word)
size_of::<&[u8]>()     // 16 — fat pointer (data ptr + length)
```

`size_of` on the reference type measures the **pointer itself**.

| Expression | type | value |
|---|---|---|
| `size_of_val(&a)` | `&[u8; 4]` → `[u8; 4]` | 4 |
| `size_of_val(&a[1..3])` | `&[u8]` → `[u8]` | 2 |
| `size_of::<&[u8; 4]>()` | the pointer type | 8 |
| `size_of::<&[u8]>()` | the pointer type | 16 |

The first two ask "how big is the data?". The last two ask "how big is the pointer?"

That is exactly why `&a[1..4]` is required — `[u8]` is unsized (no known size at compile time), but `&[u8]` is a known-size fat pointer the compiler can place on the stack.

### The big picture

| Problem | C habit | Rust way | Why |
|---|---|---|---|
| Zero-initialised array | `int buf[256] = {0}` | `let buf: [u16; 256] = [0; 256]` | Repeat expression, no memset |
| Slice of existing array | `int *p = &arr[1]` | `let s = &a[1..4]` | `[]` returns unsized `[T]`, `&` makes it `&[T]` |
| Check slice size | `sizeof` on array | `size_of_val(&s)` | Measures pointee, not pointer |

**The core lesson.** Rust gives you the same low-level control as C (zero-cost slices, no hidden copies) but forces you to be explicit about unsized vs sized types. The compiler's error messages are actually good here — when it says `[i32]` is unsized, it is telling you "you need a reference or a Vec, not a bare slice value." Once you internalise that, `&a[1..4]` becomes instinct.
