+++
title = "Rust struct types — regular, tuple, and unit"
date = 2026-07-02
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Learning", "Structs", "Tuple Structs"]
+++
---
<br>

## (*Eng*) Rust struct types — regular, tuple, and unit

Rust gives you three ways to define a struct. Each exists because sometimes naming a field is helpful, sometimes the position is enough, and sometimes you just need a named thing with no data at all.

```rust
struct Regular {
    red: u8,
    green: u8,
    blue: u8,
}

struct Tuple(u8, u8, u8);

struct Unit;
```

### Regular struct — named fields

The standard form. Every field has a name and a type.

```rust
struct ColorRegularStruct {
    red: u8,
    green: u8,
    blue: u8,
}

let green = ColorRegularStruct {
    red: 0,
    green: 255,
    blue: 0,
};
println!("{}", green.red);  // 0
```

Named fields make the code self-documenting. When you see `green.red`, you know exactly what it is.

### Tuple struct — positional fields

Fields have positions instead of names. Access them with `.0`, `.1`, `.2` — same syntax as tuples.

```rust
struct ColorTupleStruct(u8, u8, u8);

let green = ColorTupleStruct(0, 255, 0);
println!("{}", green.0);  // 0
```

The difference from a bare tuple `(u8, u8, u8)` is that `ColorTupleStruct` is its own type. The compiler will not let you pass a `ColorTupleStruct` where a `(u8, u8, u8)` is expected, or confuse it with another tuple struct that happens to have the same field types.

```rust
struct Rgb(u8, u8, u8);
struct Hsv(f32, f32, f32);

let c = Rgb(255, 0, 0);
let d = Hsv(0.0, 1.0, 0.5);
// c = d;  // ❌ type mismatch — Rgb vs Hsv
```

### The C embedded analogy

In C, you use `struct` for named fields and raw arrays for positional data:

```c
// Named fields — like regular struct
struct sensor {
    uint32_t raw;
    uint8_t channel;
};

// Positional — like tuple struct
uint8_t rgb[3] = {0, 255, 0};
rgb[0];  // 0 — positional, but unchecked
```

The difference: `struct ColorTupleStruct(u8, u8, u8)` is a **named type** even though its fields are positional. A C array `uint8_t[3]` has no type identity — you can pass it anywhere that accepts any `uint8_t*`. A tuple struct stops you from mixing up RGB values with, say, accelerometer readings that happen to also be three `u8`s.

| C concept | Rust equivalent | Type safety |
|---|---|---|
| `struct { uint8_t r, g, b; }` | Regular struct with named fields | ✅ Field access by name |
| `uint8_t arr[3]` with positional convention | Tuple struct with positional fields | ✅ Named type, won't mix with other 3-byte tuples |
| `typedef struct {} Empty` zero-size marker | Unit struct | ✅ Named type, zero bytes |

### Unit struct — the zero-size marker

```rust
struct UnitStruct;
```

A struct with no fields. It occupies zero bytes at runtime. The compiler knows it exists, but it carries no data.

```rust
let unit = UnitStruct;
println!("{:?}s are fun!", unit);  // UnitStructs are fun!
```

**Embedded use cases.** Unit structs are used as type-level markers:

- **Type-state pattern** — `struct Connected;` and `struct Disconnected;` as types on a handle. The compiler can enforce that you can't write to a disconnected port.
- **Trait implementations** — when you need a type to implement a trait but have no data to store, a unit struct is the zero-overhead way.

```rust
trait Register {
    fn address(&self) -> u32;
}

struct AdcConfig;
impl Register for AdcConfig {
    fn address(&self) -> u32 { 0x4001_2000 }
}

struct DmaConfig;
impl Register for DmaConfig {
    fn address(&self) -> u32 { 0x4002_0000 }
}
```

### Which one to use

| Situation | Struct type |
|---|---|
| Each field has a distinct meaning | Regular struct with named fields |
| Position alone conveys the meaning (RGB, XY coordinates) | Tuple struct |
| You need a distinct type with no data (type-state, trait marker) | Unit struct |
| Quick one-off grouping within a function | Bare tuple `(u8, u8, u8)` |

### The core lesson

The three struct forms are not redundancy. They give you different levels of **type identity**:

- A bare tuple `(u8, u8, u8)` is anonymous — you can pass it anywhere that shape is accepted.
- A tuple struct `Color(u8, u8, u8)` is a named type — the compiler enforces intent.
- A regular struct `Color { r, g, b }` is named **and** self-documenting — the compiler enforces intent and the code tells you what each field means.

Pick the least amount of ceremony that still makes wrong code look wrong to the compiler.
