+++
title = "Rust Option patterns and unwrap pitfalls"
date = 2026-07-09
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Option", "unwrap", "if let", "while let"]
+++

---
<br>

## (*Eng*) Rust Option patterns and unwrap pitfalls

`Option<T>` is Rust's explicit answer to missing data. It replaces null-pointer exceptions, sentinel return values, and unchecked error codes with a type-level contract. Once you treat `Option` as an enum instead of `Nullable`, the idioms `if let`, `while let`, `unwrap`, and nested matching become mechanical.

> Missing data becomes an explicit branch, not a hidden assumption.

### Some and None

```rust
enum Option<T> {
    Some(T),
    None,
}
```

Use it when a value may be absent.

```rust
let sensor: Option<u16> = None;
let cached: Option<u16> = Some(1200);
```

In embedded terms, `None` is a missing ADC sample, an empty queue slot, or a GPIO read that failed. `Some(value)` is a valid reading you can trust.

### if let: one pattern match

`if let` runs a block only if the value matches one variant.

```rust
let maybe = Some("rustlings");

if let Some(word) = maybe {
    println!("{word}");
}
```

Behind the scenes, this is shorthand for `match`. When the pattern matches, Rust binds the inner value to `word`. When it does not match, the block is skipped.

Use `if let` when only one case matters and the others are just noise. It is the Rust equivalent of pattern checking before dereferencing.

### while let: repeat while matching

`while let` repeats as long as the pattern keeps matching. It stops when the pattern breaks.

```rust
while let Some(Some(integer)) = values.pop() {
    println!("{integer}");
}
```

Each call to `pop()` returns `Option<Option<i8>>`. The first `Some` confirms the vector still contains items. The second `Some` confirms the item is a real integer and not `None`. Both `None` cases stop the loop automatically.

Use `while let` for stream-like cleanup: draining queues, unwrapping sequences, or consuming nested optional containers.

### unwrap: blind extraction

`unwrap()` immediately extracts the inner value from `Some`. If the value is `None`, the program panics.

```rust
let temp = sensor.read().unwrap();
```

In C terms, `unwrap()` is dereferencing a pointer with no NULL check. It is concise, but it trades safety for brevity. A panic in production is the same as a hard fault from a NULL dereference.

### Better alternatives to unwrap

| Situation | Prefer |
|:--|:--|
| Use only if present | `if let Some(x) = opt { ... }` |
| Return all variants explicitly | `match opt { ... }` |
| Needs a safe fallback | `unwrap_or(default)` |
| Stop on first failure | `?` when returning `Result` |

```rust
if let Some(temp) = sensor.read() {
    process(temp);
} else {
    handle_timeout();
}
```

```rust
let temp = sensor.read().unwrap_or(25);
```

### Nested option: layered unwrapping

When a container stores `Option<T>` values, every access adds an outer `Option` layer. `Vec<Option<i8>>::pop()` returns `Option<Option<i8>>`.

Correct handling is matching both layers in the pattern, not unwrapping manually.

```rust
while let Some(Some(x)) = values.pop() {
    assert_eq!(x, cursor);
    cursor -= 1;
}
```

This skips `None` values inside the container and exits cleanly when the vector empties. Attempting to unwrap the inner layer separately will crash on `None`.

> Match the shape of the data. Do not fight layers with `unwrap()`.

### Use cases

**Missing sensor reading**

```rust
let sample: Option<f32> = adc.read();

if let Some(v) = sample {
    log(v);
} else {
    log_timeout();
}
```

**Parsing with fallback**

```rust
let level = parse_level(raw_input).unwrap_or(0);
```

**Empty queue draining**

```rust
while let Some(frame) = rx_queue.pop() {
    process(frame);
}
```

### Common mistakes

**Unwrapping inside a loop**
If a `Vec<Option<T>>` contains `None`, unwrapping blindly crashes. Match nested patterns instead.

**Treating Option like a pointer**
`Option<T>` is an enum, not a nullable reference. It demands an explicit branch; the compiler will remind you until you provide one.

**Using unwrap when None is frequent**
Every `unwrap()` is a potential panic site. Replace it with `unwrap_or`, `if let`, or `match` once the failure path becomes real.

### Reference checklist

- `Some(value)` means data exists.
- `None` means no data.
- `if let Some(x) = opt` executes one branch.
- `while let ... = stream` repeats until the pattern fails.
- `unwrap()` is for known-safe cases only.
- For layered containers, match nested patterns like `Some(Some(x))`.
- For fallbacks, use `unwrap_or(default)`.
