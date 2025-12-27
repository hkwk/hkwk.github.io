---
author: HKL
categories:
- Default
date: "2025-12-27T10:51:00+08:00"
slug: windows-gnu-ccsds-aec-fix
draft: false
tags:
- Programming
- Rust
title: Windows (x86_64-pc-windows-gnu) 上启用 GRIB2 5.0=42 (CCSDS/AEC) 解码的修复记录
---

# Windows (x86_64-pc-windows-gnu) 上启用 GRIB2 5.0=42 (CCSDS/AEC) 解码的修复记录

## 背景

我们的 GRIB2 预览工具/GUI 使用 Rust 的 `grib` crate 解析并解码 GRIB2。
当输入文件采用 **GRIB2 code table 5.0 = 42**（Data Representation Template 42，CCSDS/AEC packing）时，默认构建会报“不支持”。

为支持该 packing，需要启用：

- `grib/ccsds-unpack-with-libaec` → 依赖 `libaec-sys`（会触发 native 构建 + bindgen）

本项目中将其做成 opt-in feature：

- `--features grib-ccsds`

## 现象与问题链路

1. 运行预览时提示 template 42 不支持。
2. 开启 `--features grib-ccsds` 后，开始编译 `libaec-sys`：
   - `libaec-sys` 先通过 CMake 构建 C 库（libaec）。
   - 然后通过 bindgen 生成 Rust FFI 绑定，这一步需要加载 `libclang.dll`。
3. 在 Windows GNU 工具链 + 混合环境（MSYS2/Conda/System32 DLL）下，出现两类问题：
   - **MSYS2 工具链组件崩溃/异常**（例如 assembler 依赖 DLL 冲突）。
   - **bindgen/clang-sys 无法加载 `libclang.dll`**，报 `LoadLibraryExW failed`。

## 根因定位（关键点）

### 1. `libclang.dll` 能找到，但加载失败

即使 `LIBCLANG_PATH=C:\msys64\ucrt64\bin` 且该目录存在 `libclang.dll`，仍可能加载失败。

实际原因通常不是“找不到 libclang”，而是：

- `libclang.dll` 的依赖 DLL（例如 `zlib1.dll` 等）被 Windows loader 从 **System32 / Conda** 等位置优先解析到了不兼容版本。

我们用 Win32 API 复现时发现：

- `LoadLibraryW(libclang.dll)` 失败（错误码 127，常见于依赖或导出不匹配）
- 但 `LoadLibraryExW(libclang.dll, LOAD_WITH_ALTERED_SEARCH_PATH)` 成功

这说明是“依赖 DLL 搜索路径/顺序”问题。

### 2. `libaec-sys` 在 windows-gnu 上链接库名错误

`libaec-sys` 原始 build script 在 Windows 直接链接 `aec-static`（更像 MSVC 产物名），
但在 MinGW/MSYS2 下实际安装产物是 `libaec.a`（应当链接为 `aec`）。

因此出现：

- `could not find native static library aec-static`

## 代码级修复

### A. 修复 clang-sys 加载方式（解决 libclang 加载失败）

做法：在 Windows 下，加载 `libclang.dll` 时改用 `LOAD_WITH_ALTERED_SEARCH_PATH`，
使依赖 DLL 更倾向于从 `libclang.dll` 所在目录解析。

对应修改：

- `vendor/clang-sys-1.8.1/src/link.rs`

并通过 `Cargo.toml`：

- `[patch.crates-io] clang-sys = { path = "vendor/clang-sys-1.8.1" }`

### B. 修复 libaec-sys 在 windows-gnu 的链接名（解决 aec-static 缺失）

做法：根据 `CARGO_CFG_TARGET_ENV` 区分：

- `windows-msvc`：链接 `aec-static`
- `windows-gnu`：链接 `aec`

对应修改：

- `vendor/libaec-sys-0.1.3/build.rs`

并通过 `Cargo.toml`：

- `[patch.crates-io] libaec-sys = { path = "vendor/libaec-sys-0.1.3" }`

## 使用方式

- 仅构建 GUI（不需要 CCSDS/AEC）：

  `cargo run`

- 解码 template 42 并生成预览（需要 native 依赖）：

  `cargo run --features grib-ccsds --bin grib_preview -- data.grib2`

## 备注（环境一致性建议）

- 尽量保持 `cmake` / `gcc` / `make` / `clang` 来自同一套 MSYS2 UCRT64 环境。
- 避免 PATH 中 Conda/System32 的 zlib/libzstd 等 DLL 抢先被加载导致崩溃或 LoadLibraryExW 失败。
- 后期正在考虑使用rust重写一下 `libaec` 模块