+++
title = "Rust declaration, definition and the enum error pattern"
date = 2026-07-17
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Ownership", "Modules", "Error Handling"]
+++
---
<br>

## (*Eng*) Rust declaration, definition and the enum error pattern

Coming from C, you are used to two files for every function: a header for the contract and a source file for the implementation. Rust drops that split. The definition is the declaration, the module system replaces headers, and error handling leans on enums instead of integer codes.

This post covers both topics together because they are the two places where Rust's "no separate declaration" model shows up most clearly: traits as interfaces and enums as error contracts.

> In Rust, the compiler sees the whole crate at once. That single fact removes the need for headers and forward declarations.

### C model versus Rust model

In C, you write a header to tell other translation units what exists:

```c
// foo.h
uint32_t adc_read(uint8_t channel);
```

Then you write the source file to tell the compiler how it works:

```c
// foo.c
uint32_t adc_read(uint8_t channel) {
    return HAL_ADC_GetValue(channel);
}
```

The header is a contract enforced by convention. If you change the source and forget to update the header, you get silent mismatch bugs. You also need forward declarations when functions call each other.

Rust folds both into one definition:

```rust
pub fn adc_read(channel: u8) -> u32 {
    HAL_ADC_GetValue(channel)
}
```

The compiler reads the entire crate together, so it knows the full signature before compiling call sites. You can even call a function before its textual definition:

```rust
fn main() {
    adc_read(0); // works even though adc_read is defined below
}

fn adc_read(channel: u8) -> u32 {
    42
}
```

Because the compiler sees the whole crate, there is no mechanical need for forward declarations.

### Traits replace header interfaces

If you want to share a contract across multiple types without sharing implementation, you use a trait:

```rust
trait Adc {
    fn read(&self, channel: u8) -> u32; // declaration only
}
```

Any type that implements `Adc` must provide a body. This is the closest thing Rust has to a C header with pure virtual functions. Unlike C, traits can also provide default implementations:

```rust
trait Adc {
    fn read(&self, channel: u8) -> u32 {
        0 // default implementation
    }
}
```

Implementors inherit the default unless they override it. This is similar to a base class method in C++, but the choice of whether to override is left to the implementor.

### `extern` for foreign-function declarations

When you need to call into C or another ABI, `extern "C"` acts like an `extern` declaration in a header:

```rust
extern "C" {
    fn HAL_ADC_Start(hadc: *mut ADC_Type) -> u32;
}
```

You are declaring that the function exists elsewhere. Rust will link it at build time. The body is never written in Rust. This is the exact equivalent of `extern uint32_t HAL_ADC_Start(ADC_Type *hadc);` in a C header.

### `mod` and `use` replace `#include`

Instead of `#include "adc.h"`, Rust declares modules and brings names into scope:

```rust
// lib.rs
pub mod adc;

// main.rs
use mycrate::adc;
adc::read(channel);
```

Visibility is controlled with `pub`, but there is still no separate declaration file. The module path is the namespace. If a function is private to the module, it does not appear in the public contract at all, which is stronger than a static function in a `.c` file.

### Enums replacing integer error codes

C error handling often looks like this:

```c
int parse_positive(const char *s, uint64_t *out) {
    char *end;
    long v = strtol(s, &end, 10);
    if (errno != 0) return -1;     // parse error
    if (v <= 0) return -2;         // range error
    *out = (uint64_t)v;
    return 0;                      // success
}
```

The caller must remember what `-1` and `-2` mean. There is no type-level guarantee that the return value was checked.

Rust models both outcomes as an enum:

```rust
enum ParsePosNonzeroError {
    ParseInt(ParseIntError),
    Creation(CreationError),
}
```

`ParseInt(ParseIntError)` wraps the standard parsing failure. `Creation(CreationError)` wraps the range-validation failure. Callers match on the variant instead of decoding magic numbers:

```rust
match PositiveNonzeroInteger::parse(input) {
    Ok(n) => use(n),
    Err(ParsePosNonzeroError::ParseInt(_)) => retry_or_log(),
    Err(ParsePosNonzeroError::Creation(_)) => reject_range(),
}
```

The enum variant names are the documentation. The type system enforces that every branch is handled.

### Enum variant constructors are free functions

When a variant carries data, Rust gives you a constructor function automatically:

```rust
enum ParsePosNonzeroError {
    ParseInt(ParseIntError),
    Creation(CreationError),
}
```

- `ParsePosNonzeroError::ParseInt` is a function `fn(ParseIntError) -> ParsePosNonzeroError`
- `ParsePosNonzeroError::Creation` is a function `fn(CreationError) -> ParsePosNonzeroError`

You pass them around like any other function pointer:

```rust
s.parse().map_err(ParsePosNonzeroError::ParseInt)?;
```

`map_err` receives the constructor and calls it for you when an error occurs. In C terms, this is a factory function for error codes, but generated by the compiler instead of written by hand.

### Embedded analogy

Think of a peripheral driver library. In C, the header file is the public API contract and the `.c` file is the private implementation. You must keep them in sync manually. In Rust, a `trait` is the contract and an `impl` block is the implementation. They live adjacent or in the same file. The compiler enforces consistency every build.

For errors, C status codes are manual bit flags or magic negative integers. Rust enums are tagged unions: each variant carries its own payload and the compiler forces you to handle every case. That is the difference between "check the errno docs" and "the compiler will not let you ignore the failure."

### Practical rules

| C pattern | Rust equivalent |
|:--|:--|
| Header file with declarations | `trait` or direct function definitions |
| `static` function in `.c` | `fn` without `pub` in a module |
| `extern` declaration | `extern "C" { fn ...; }` |
| `#include "foo.h"` | `use crate::foo;` or `mod foo;` |
| Magic integer error codes | `enum Error { ... }` |
| Base class with virtual methods | `trait` with method declarations |
| Forward declaration | Not needed in Rust |

### Reference checklist

- Function definition is its own declaration. No headers.
- Traits are the type-level contracts that replace header interfaces.
- `extern "C"` is for foreign-function declarations only.
- `mod` and `use` structure the namespace without duplicate files.
- Enum variants with payloads generate constructor functions automatically.
- Use enums for errors so callers can match on concrete failure modes.
- The compiler enforces declaration/definition consistency every build.
