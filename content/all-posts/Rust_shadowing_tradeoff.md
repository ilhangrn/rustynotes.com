+++
title = "Rust shadowing tradeoff"
date = 2026-06-25
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Learning", "Shadowing"]
+++
---
<br>

## (*Eng*) Rust shadowing tradeoff

Shadowing in Rust feels wrong when you come from C.

```rust
let number = 3;
let number = number + 2;
```

In C this is an error for good reason. But Rust shadowing is not two things with the same name. The first `number` is gone. You can never reference it again. The compiler enforces this.

> Shadowing is replacing a binding, not duplicating a name.

**Why it exists.**

Consider reading and validating an ADC value in embedded code.

```c
// C approach: different names for each stage
uint32_t adc_raw = adc_read(ADC_CHANNEL_0);
int validated = validate_range(adc_raw);
if (validated) {
    uint16_t adc_filtered = lowpass_filter(adc_raw);
    // Which variable do I use here? adc_raw or adc_filtered?
    // Both exist, both are in scope. If I pick wrong, subtle bug.
}
```

```rust
// Rust approach: same name, old one disappears
let adc_val = adc_read(ADC_CHANNEL_0);
let adc_val = validate_range(adc_val)?;  // old adc_val gone
let adc_val = lowpass_filter(adc_val);   // validated value gone
// Only the filtered value exists — can't accidentally use the raw one
```

Each shadow step consumes the previous binding. You cannot access the raw register value anymore. The compiler prevents the bug.

Same idea for type conversion:

```rust
let input = get_string();
let input = input.trim();
let input = input.parse::<f32>()?;
```

In C you end up with `char* input_str` then `float input_float` — two names for the same logical thing. Shadowing lets you keep `input` through the whole pipeline.

**But there is a real cost.**

During debug, different names help a lot. Your watch window shows every stage of the pipeline. `adc_raw`, `adc_filtered`, `adc_scaled` — each has its own address and lifetime. You can inspect how the filter changes the raw values. With shadowing, earlier values are dropped and gone. You cannot see them anymore.

This is one of those cases where Rust's compile-time safety (prevent bugs before they happen) bumps into embedded reality (you will stare at a watch window at 3am).

**The pragmatic take.**

- Use distinct names for multi-stage transforms where you want to inspect intermediate values during debug. `raw → filtered → scaled → output`
- Use shadowing for short, mechanical conversions where the intermediate state is noise. `input.trim().parse()`, temporary `mut` borrows, one-liner transformations.

There is no rule. Your debugger workflow is a valid reason to prefer distinct names. The compiler will not complain either way.
