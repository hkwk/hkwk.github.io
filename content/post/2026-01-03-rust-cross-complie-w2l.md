---
author: HKL
categories:
- Default
date: "2026-01-03T22:12:00+08:00"
slug: rust-cross-complie-w2l
draft: false
tags:
- Programming
- Rust
title: Cross-compilation (Windows → Linux using Zig)
---

This document explains a small, **project-independent** way to cross-compile Rust targets (for example, `x86_64-unknown-linux-musl`) from Windows using Zig as a system linker/sysroot. It also covers alternative tools (`cargo-zigbuild`, `cross`) and troubleshooting tips.

## Summary ✅
- Create Zig wrapper scripts (linker/ar) in a folder on `PATH` and add a global Cargo config (`%USERPROFILE%\.cargo\config.toml`).

example Project here:

```powershell
cargo build --release --target x86_64-unknown-linux-musl

PS D:\workspace\rust\rust-cross-build-w2l-script> cargo build --release --target x86_64-unknown-linux-musl
   Compiling cfg-if v1.0.4
   Compiling libc v0.2.178
   Compiling zerocopy v0.8.31
   Compiling getrandom v0.2.16
   Compiling rand_core v0.6.4
   Compiling ppv-lite86 v0.2.21
   Compiling rand_chacha v0.3.1
   Compiling rand v0.8.5
   Compiling helloworld v0.1.0 (D:\workspace\rust\rust-cross-build-w2l-script)
    Finished `release` profile [optimized] target(s) in 3.62s
PS D:\workspace\rust\rust-cross-build-w2l-script> 

```

---

## Prerequisites 🔧
- Install Zig and put it on your `PATH` (https://ziglang.org/download).
- Rust toolchain installed with `rustup`.
- Add the target you want to build:

```powershell
rustup target add x86_64-unknown-linux-musl
```

---

Global Cargo config + Zig wrappers (recommended for control)

1. Create a central folder for wrappers (example): `C:\tools\zig-cross\bin` and add it to your Windows `PATH`.

2. Create the wrapper scripts. Example Windows CMD wrapper `zig-x86_64-linux-musl.cmd`:

```bat
@echo off
REM Acts as a linker for cargo
zig.exe cc -target x86_64-linux-musl -static %*
```

Example `zig-ar.cmd`:

```bat
@echo off
zig.exe ar %*
```

(If you use PowerShell, you can make `.ps1` wrapper versions; for WSL/Linux use a small shell script calling `zig cc -target ...`.)

3. Create a **global Cargo config** (applies to all projects) at `%USERPROFILE%\.cargo\config.toml` with the target section:

```toml
[target.x86_64-unknown-linux-musl]
linker = "zig-x86_64-linux-musl"
ar = "zig-ar"
rustflags = ["-C", "target-feature=+crt-static"]
```

- The `linker` and `ar` must be resolvable on `PATH` and match the wrapper filenames.

4. Build in any project:

```powershell
cargo build --release --target x86_64-unknown-linux-musl
```

---

## Troubleshooting & Tips ⚠️
- "linker not found": ensure wrapper name and location are on `PATH` and match the `linker` setting.
- Static linking: when targeting `*-musl`, you may need `-C target-feature=+crt-static` (`rustflags`) to avoid dynamic glibc.
- If Rust crates require system libraries, ensure they are available in the Zig sysroot or avoid those crates.
- Add other targets by creating additional wrapper scripts and Cargo `target.*` sections.

---

If you'd like, use can use the sample scripts and a copyable `config.toml` in this repo.

An example project is placed [github.com/hkwk/rust-cross-build-w2l-script](https://github.com/hkwk/rust-cross-build-w2l-script).
