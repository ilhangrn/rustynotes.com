+++
title = "Rust traits: shared behavior"
date = 2026-07-01
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Traits", "Generics"]
+++
---
<br>

## (*Eng*) Rust traits: shared behavior

Traits define shared behavior across types. A trait declares methods that implementing types must provide. Other languages call this an interface. Rust's twist: traits can also provide default method implementations.

> Traits let you write generic code over any type that implements a behavior.

### Defining a trait

```rust
trait Summary {
    fn summarize(&self) -> String;

    fn preview(&self) -> String {
        format!("Preview: {}", self.summarize())
    }
}
```

Types that implement `Summary` must provide `summarize`. They get `preview` for free unless they override it.

### Implementing a trait

```rust
struct Article {
    title: String,
    body: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}", self.title)
    }
}
```

```rust
let article = Article { title: String::from("Rust traps"), body: String::from("...") };
println!("{}", article.summarize());
println!("{}", article.preview());
```

### Using traits as bounds

Generics accept trait bounds to constrain types.

| Pattern | Meaning |
|:--|:--|
| `T: Summary` | `T` implements `Summary` |
| `fn notify<T: Summary>(item: &T)` | generic over any `Summary` type |
| `impl Summary for MyType` | concrete implementation |

```rust
fn notify(item: &impl Summary) {
    println!("Breaking: {}", item.summarize());
}
```

The `impl Trait` syntax is shorthand for a generic bound. It reads naturally: `item` is some type that implements `Summary`.

### Default implementations reduce duplication

When many types share the same logic, put it in the trait body.

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(no summary)")
    }
}
```

Callers still override when needed, but boilerplate disappears.

### Difference from C++ concepts or Java interfaces

- Traits can provide defaults directly, not just declarations.
- Trait bounds are part of the type system, not a separate annotation layer.
- A type can implement a trait only if either the trait or the type is local to your crate. This prevents orphan conflicts.

> Core lesson: traits are Rust's main abstraction for polymorphism. They separate "what a type can do" from "what a type is."
