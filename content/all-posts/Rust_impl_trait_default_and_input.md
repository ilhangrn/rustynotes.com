+++
title = "Rust impl, trait defaults and passing traits as input"
date = 2026-07-20
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Traits", "Generics"]
+++
---
<br>

## (*Eng*) Rust impl, trait defaults and passing traits as input

In C, you separate interface from implementation: a header says what exists, a source file says how it works. Rust collapses that split. A `trait` declares the interface, `impl` provides the implementation, default methods live inside the trait, and `impl Trait` lets callers pass any compatible type into a function without exposing generics.

This post connects four pieces that show up together in rustlings `traits4.rs`: `trait`, default method bodies, `impl Trait` for types, and `impl Trait` as function input.

> A `trait` is a contract. An `impl` block is the implementation of that contract. `impl Trait` as a parameter is Rust's way of saying "I accept any type that satisfies this contract."

### Trait declaration with a default method

A trait can declare methods without giving them a body, or it can provide a default body that implementors inherit automatically:

```rust
trait Licensed {
    fn licensing_info(&self) -> String {
        "Default license".to_string()
    }
}
```

`Licensed` declares one method, `licensing_info()`, and supplies a default implementation. Any type that writes `impl Licensed for MyType {}` inherits that default. If it wants a custom message, it overrides just that one method:

```rust
impl Licensed for SomeSoftware {
    fn licensing_info(&self) -> String {
        "GPLv3".to_string()
    }
}
```

This is similar to a C++ base class with a virtual method and a default definition, except Rust traits do not require inheritance. You opt in by implementing the trait.

### impl blocks give behavior to concrete types

`impl Trait for Type` attaches a trait to a concrete type. The simplest form inherits everything from the trait:

```rust
struct SomeSoftware;
struct OtherSoftware;

impl Licensed for SomeSoftware {}
impl Licensed for OtherSoftware {}
```

Both structs get the default `licensing_info()`. Neither struct declares any data; they only opt into the contract. That is enough for the compiler to treat them as "something that knows how to describe its license."

### Passing traits as function input

When a function only needs to call methods from a trait, it does not need to know the concrete type. `impl Licensed` expresses exactly that:

```rust
fn compare_license_types(software1: impl Licensed, software2: impl Licensed) -> bool {
    software1.licensing_info() == software2.licensing_info()
}
```

The function signature means:

- `software1` can be any type that implements `Licensed`.
- `software2` can be any other type that implements `Licensed`.
- The two types do not need to be the same.

This is why `compare_license_types(SomeSoftware, OtherSoftware)` compiles: each argument satisfies the contract independently. The caller chooses the concrete type; the function only depends on the trait's interface.

### Behind the syntax: hidden generics

`impl Licensed` is syntactic sugar for an anonymous generic parameter:

```rust
fn compare_license_types<T: Licensed, U: Licensed>(
    software1: T,
    software2: U,
) -> bool {
    software1.licensing_info() == software2.licensing_info()
}
```

`T` and `U` are separate types. Each gets its own trait bound. The sugar hides the type parameter names, which is useful when the function does not need to refer to the concrete type anywhere else.

If you want both arguments to be the same concrete type, you would use one named parameter:

```rust
fn compare_same<T: Licensed>(a: T, b: T) -> bool {
    a.licensing_info() == b.licensing_info()
}
```

Now `compare_same(SomeSoftware, OtherSoftware)` would fail, because `SomeSoftware` and `OtherSoftware` are different types. The constraint changed the contract.

### Where defaults matter

Default methods in traits reduce boilerplate for common behavior. If ten drivers in an embedded HAL share the same timeout or baud-rate fallback, the trait can provide the default and only override it in special cases:

```rust
trait Uart {
    fn baud_rate(&self) -> u32 {
        115_200 // default fallback
    }

    fn write(&self, data: &[u8]);
}
```

Every `Uart` implementor must define `write()`. It may ignore `baud_rate()` and inherit the default, or override it for a faster peripheral. Callers can rely on `baud_rate()` existing without each implementor rewriting it.

### Using impl Trait in return position

The same `impl Trait` sugar works for return types, with one restriction: the concrete type must be unambiguous to the caller:

```rust
fn licensed_software() -> impl Licensed {
    SomeSoftware
}
```

This says "the function returns some single concrete type that implements `Licensed`." The caller does not know which one, but the compiler guarantees it implements the trait. You cannot return two different concrete types conditionally from the same function with `impl Licensed`; for that you need an `enum` or a trait object.

### Common pitfalls

- **Forgetting default inheritance:** If a trait provides a default method and you write `impl Trait for Type {}` without overriding it, you still get the default. Do not add an empty method body expecting to replace it.
- **Mixing `impl Trait` and generics unnecessarily:** `impl Trait` as input is shorthand when the function needs the trait but does not need to name the concrete type. If the function needs multiple methods, associated types, or trait bounds with extra constraints, a named generic is clearer.
- **Assuming input `impl Trait` ties arguments together:** Each `impl Trait` parameter introduces its own hidden type slot. If you want both arguments forced to the same type, use one generic parameter.

### Embedded analogy

Think of a driver interface board. The `trait` is the socket pinout: any chip that matches it can plug in. The `impl` block is the actual driver for one specific chip. Default methods are like a built-in safety timeout that every driver inherits unless it overrides the value. `impl Licensed` as a parameter is the board saying "give me any driver that follows this pinout" instead of hardcoding one part number.

### Practical rules

| Pattern | Meaning |
|:--|:--|
| `trait Foo { fn bar(&self); }` | Contract with required method |
| `trait Foo { fn bar(&self) { default } }` | Default implementation |
| `impl Foo for Type {}` | Attach trait to concrete type |
| `fn takes(x: impl Foo)` | Accept any type implementing `Foo` |
| `fn returns() -> impl Foo` | Return one concrete type implementing `Foo` |
| `fn generic<T: Foo>` | Named generic when type must be explicit |

### Reference checklist

- A `trait` can declare methods, provide defaults, or mix both.
- Default methods reduce repetition for cross-cutting behavior.
- `impl Trait` as input means the caller chooses the concrete type.
- Each `impl Trait` parameter is independent unless you use one generic parameter explicitly.
- `impl Trait` in return position hides one concrete return type.
- Use an `enum` or trait object when you need to return multiple concrete types behind one interface.
