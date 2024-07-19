+++
title = "Embedded rust entry"
date = 2024-07-19
[taxonomies]
categories=["Guides"]
tags=["Rust", "Environment", "Embedded", "Guide"]
+++
---
<br>

## (*Eng*) Embedded rust entry
> It is already here. Future is here, i don't want to miss future.

> To start we need to install rust on our computer. [Install rust from here](https://www.rust-lang.org/tools/install)

> Then we call `rustup update` command to get most updated version. Because it gets release every 6 weeks.

> Compiler is `rustc`, package manager is `cargo`.

> We will use `vscode`, it is not the fastest ide but so easy. Then we will install `rust anaylzer` for our coding experience.

> Rustc compiler is front end of LLVM. Which is making our job possible.

> Rust runs on most of microcontroller families. Has a great support in Arm Cortex and Risc 5. We will use devkit. Because they have one extra chip onboard for usb debugging.

> Now how we build for this chips? We are going to use power of cross compilation. We will compile and run in our computer. But to compile same code in our device we need compile it again for target. We will define compile target.

> `<arch><sub>-<vendor>-<sys>-<enc>`
- arch = x86_64, i386, arm, ...
- sub = [ex. arm] v5, v6, v7m, ...
- vendor = [optional] pc, apple, ibm, ...
- sys = none, linux, win32, darwin, ...
- env = eabi, gnu, elf, ...

> Then we will add it as target by `rustup target add`

> For `microbit dev board v2`, we have `nRF52833` with arm cortex m4. When we check from arm website, we see it has `Armv7E-M` arch. Then we visit Rust platform support page to see target specs. When we search for Armv7E we see target will be `thumbv7em-none-eabihf` for our development. Our command will be `rustup target add thumbv7em-none-eabihf`.

> By running `rustup show` we can see our targets.

> To create project we run `cargo new myDemo`

> It will create the project directory. We can reach by `cd mydemo`

> It has `Cargo.toml` and `Cargo.lock` files for dependency. And we will have `src` directory there which has `main.rs` file and our next future code files.

> Cargo will run check in background to control our dependencies.

---

> We will code our application. We will add `#![no_std]` and `#![no_main]` for embedded.

> To see our code placement in chip memory we need module named `cortex-m-rt`. We can go to `crates.io` website and reach all related details. It says 
```
    cortex-m-rt

    Startup code and minimal runtime for Cortex-M microcontrollers
```

> Has a reference to technical documents for this crate as `https://docs.rs/cortex-m-rt/0.7.3/cortex_m_rt/`
It says:
```
Startup code and minimal runtime for Cortex-M microcontrollers

This crate contains all the required parts to build a no_std application (binary crate) that targets a Cortex-M microcontroller.

---
Features
---

This crates takes care of:

    The memory layout of the program. In particular, it populates the vector table so the device can boot correctly, and properly dispatch exceptions and interrupts.

    Initializing static variables before the program entry point.

    Enabling the FPU before the program entry point if the target is thumbv7em-none-eabihf.

This crate also provides the following attributes:

    #[entry] to declare the entry point of the program
    #[exception] to override an exception handler. If not overridden all exception handlers default to an infinite loop.
    #[pre_init] to run code before static variables are initialized

This crate also implements a related attribute called #[interrupt], which allows you to define interrupt handlers

```

> We will add it by `cargo add cortex-m-rt`. Then we will create a `memory.x` file in project folder and add specifications below.For the NRF52833 with details found from
memory map from datasheet.

```
MEMORY
{
  FLASH : ORIGIN = 0x00000000, LENGTH = 512K
  RAM : ORIGIN = 0x20000000, LENGTH = 128K
}
```

> Then we make our main code never return:
```
#![no_std]
#![no_main]

use cortex_m_tr::entry;

#[entry]
fn main() -> !{
    loop{}
}
```

> Now we need to tell to rustc to run our linker whenever it builds. So we create `.cargo` directory in project. Add our `config.toml` file like below. (To reach this command: 
`cargo rustc --target thumbv7m-none-eabi -- \ -C link-arg=-nostartfiles -C link-arg=-Tlink.x`)
```
[build]
target = "thumbv7em-none-eabihf"

[target.thumbv7em-none-eabihf]
rustflags = ["-C", "link-arg=-Tlink.x]
```

> Remember we said rustc will run the compiler in background to check the code and errors. But by default it is for our computer. We need to change it for our target. We go to `.vscode` folder and add below lines to `settings.json`.
```
{
    "rust-analyzer.check.allTargets": false,
    "rust-analyzer.cargo.target":"thumbv7em-none-eabihf"
}
```

> Our project will ask for panic handler when we try to build our bare metal empty project. We have to have a panic handler t compile. We can add it by a crate easily. It is panic-halt. It will a panic handler as infinite loop. We will add it by `cargo add panic-halt` to project then add one line to our code like:
`use panic-halt as _:`

> Now rust analyzer and cargo check both are happy.

> Now we need our tools for embedded. We call `rustup component add llvm-tools` to add llvm tools for size, binary file details and dissambly tools. Then we call `cargo install cargo-binutils` as wrapper for llvm-tools to make it easy to use.

> Now we can check size by `cargo size -- -Ax`. It will show all details about application sections and size.

> To flash it, we need one more tool. `cargo install cargo-embed`

> Then we can call `cargo embed --help` to see guide. By `cargo embed --list-chips`. It is a huge list. We can filter for our chip by `cargo embed --list-chips | grep -i nrf52833`

> Now we can embed it by `cargo embed --chip nRF52833_xxAA`

---

> For using this details in next time we will add `Embed.toml` file to our project. 
```
[default.general]
chip = "nRF52833_xxAA"
```

> Now we can embed it by `cargo embed`. We need debugger. The crate for it is `rtt-target`. (It will require `critical-section` so we will add crate `cortex-m`). We call `cargo add cortex-m` and `cargo add rtt-target`

> To enable it need to add it to our `Embed.toml` file as below.
```
[default.general]
chip = "nRF52833_xxAA"

[default.rtt]
enabled = true
```
> It is so easy to use. We init in the entry of main. Then we call `rprintln!` like below.
```
#![no_std]
#![no_main]

use cortex_m::asm::nop;
use cortex_m_tr::entry;
use panic-halt as _:
use rtt_target::{rprintln, rtt_init_print}

#[entry]
fn main() -> !{
    rtt_init_print!();
    rprintln!("Hello rusty");
    loop{
        rprintln!("loopy");
        for _ in 0..100_000{
            nop();
        }
    }
}
```

> Now we need debugging like a pro. With breakpoints, with memory check and touch. We go with `gdb` like for last 30 years.
> In linux we install by `sudo apt-get install gdb-nultiarch`
> In windows we download from arm-gnu-toolchain-downloads from arm website.

> To enable gdb we will update our Embed.toml file as below:
```
[default.general]
chip = "nRF52833_xxAA"

[default.rtt]
enabled = false

[default.gdb]
enabled = true

[enabled.reset]
halt_aferwards = true
```

> We closed rtt, and will change the code to increment a variable value and watch. It will be as below:
```
#![no_std]
#![no_main]

use cortex_m::asm::nop;
use cortex_m_tr::entry;
use panic_halt as _;

#[entry]
fn main() -> !{
    let mut x: usize = 0;
    loop{
        x += 1;
        for _ in 0..x{
            nop();
        }
    }
}
```

> we will run embed in one terminal and run gdb in another terminal. We will run gdb by `arm-none-eabi-gdb target/thumbv7...`. We will give target binary location in this.

> then it will run. We will connect to our device by `target remote : 1337`

> we can check out register values by `info registers`. It will list R0 to R12 with other registers.
  
> 