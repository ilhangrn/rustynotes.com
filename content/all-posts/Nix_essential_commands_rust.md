+++
title = "Essential Nix commands for Rust projects"
date = 2026-07-06
[taxonomies]
categories=["Study Notes"]
tags=["Nix", "Rust", "Tooling"]
+++
---
<br>

## (*Eng*) Essential Nix commands for Rust projects

Nix gives you reproducible Rust toolchains without polluting your system. The modern CLI splits responsibilities into four commands you will actually use on a daily basis. The mental model is simple: develop means write, shell means consume, run means execute, and build means package.

> Four commands, four intents: `nix develop`, `nix shell`, `nix run`, `nix build`.

### Why this matters for Rust

In C/C++ you often reach for Docker, pyenv, rustup, or a global `cargo install`. Nix replaces all of that with per-project environments. No version managers, no hidden global state. Each project declares its exact toolchain in one flake or `shell.nix`, and Nix materializes it on demand.

### nix develop: the project workspace

`nix develop` starts an interactive shell pre-configured with the exact dependencies Nix would use to build your project. It does not compile your project for you. It stops just before the build and hands control to you.

```bash
# Enter the dev shell for the current flake's default devShell
nix develop

# Or enter a named Rust dev shell
nix develop nixpkgs#rustPlatform
```

Inside `nix develop`, you get:
- The correct Rust toolchain
- Cargo, rustc, rustfmt, clippy
- Any native libraries your build needs, such as OpenSSL or SQLite

```bash
nix develop
cargo build
cargo clippy
cargo test
```

> `nix develop` is the equivalent of your embedded Makefile environment: compiler, linker flags, libraries, and headers all resolved and ready.

### nix shell: the one-off tool

`nix shell` is for software you want to run, not modify. It prepends the package's `bin` directory to `$PATH` and drops you into a shell.

```bash
# Run a single tool without installing it
nix shell nixpkgs#cargo-watch

cargo-watch -x test
```

This is also useful when you need a specific version of a tool outside your project's dev shell.

```bash
# Use a pinned cargo version
nix shell nixpkgs#cargo_2
cargo --version
```

> `nix shell` is like `extract` from a firmware image: you get the usable binary, not the whole build environment.

### nix run: execute without entering

`nix run` compiles or downloads a package and executes it directly. No manual `$PATH` management needed.

```bash
# Run a CLI tool from nixpkgs
nix run nixpkgs#cargo-audit -- audit
```

With flakes you can run project binaries directly when `package` or `apps` are defined.

```bash
# Run the default app from a flake
nix run .

# Run a specific app
nix run .#my-cli --
```

> `nix run` is the shortest path from declaration to execution. Think of it like calling a function from a library: no linker setup, just run.

### nix build: produce the artifact

`nix build` compiles the package and stores the result in the Nix store. The output path is deterministic and reproducible.

```bash
# Build the default package
nix build

# Build a specific package from nixpkgs
nix build nixpkgs#ripgrep

# Inspect where the result landed
ls -l ./result
```

For Rust projects, `nix build` can produce binaries, libraries, or even Docker images from a flake output. The result is always content-addressed: identical inputs yield identical outputs.

```bash
# Build the release binary defined in flake.nix
nix build .#my-app

# ./result/bin/my-app is now available
./result/bin/my-app --version
```

> `nix build` is your CI pipeline in local form. Same derivation, same environment, same binary every time.

### The command matrix

| Intent | Command | Stops before build? | Gives you binary? | Use case |
| Write or debug | `nix develop` | Yes | No | Dependency-heavy Rust work |
| Use a tool | `nix shell` | Yes | Yes | ad-hoc binaries |
| Run and exit | `nix run` | No | No | one-shot commands |
| Compile artifact | `nix build` | No | Yes | reproducible binaries |

### Practical Rust workflow

```bash
# 1. Enter the dev shell
nix develop

# 2. Inside, run cargo commands
cargo check --all-targets
cargo test
cargo build --release

# 3. Build the packaged artifact
exit
nix build .#my-app

# 4. Run the artifact directly
nix run .#my-app
```

The boundaries are strict by design and they consistently remove guesswork.

> Core lesson: `nix develop` writes the software, `nix shell` provides it, `nix run` uses it, and `nix build` ships it.
