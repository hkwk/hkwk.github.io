---
author: HKL
categories:
- Default
date: "2025-12-28T18:51:00+08:00"
slug: rust-aec-handoff-notes
draft: false
tags:
- Programming
- Rust
title: rust-aec v0.1.0 Handoff Notes
---

## 编写目的

- 说明 `rust-aec` crate 的使用方式、接口形态与注意事项。
- 帮助在 GRIB2 模板 5.0=42（CCSDS/AEC）等场景中，以纯 Rust 方式解码 AEC 负载数据。

## 1) Current Status (Done)

- `rust-aec` is a **pure Rust** CCSDS **121.0-B-3** Adaptive Entropy Coding (AEC) decoder, primarily targeting **GRIB2 Data Representation Template 5.0 = 42**.
- **Byte-for-byte correctness vs libaec**: verified using an oracle approach on real GRIB2 data; output matches **libaec** exactly (tests pass).
- **crates.io readiness**: `README.md`, MIT `LICENSE`, crate metadata (`repository/homepage/docs.rs`, etc.) are prepared and `cargo package` dry-run succeeds.

## 2) Key Files (should exist in the new repo)

- Crate metadata:
  - `Cargo.toml`
- Documentation:
  - `README.md` (English; includes comparison images)
  - `LICENSE` (MIT)
- Core source:
  - `src/lib.rs` (public API: `decode`, `decode_into`, `flags_from_grib2_ccsds_flags`)
  - `src/decoder.rs` (core decoder state machine + bitstream parsing + inverse preprocessing aligned to libaec)
  - `src/bitreader.rs` (MSB-first bit reader)
  - `src/params.rs` (`AecParams`/`AecFlags`)
  - `src/error.rs` (`AecError`)
- Example program:
  - `examples/decode_aec_payload.rs` (minimal: decode payload → output byte stream)
- Tests:
  - `tests/oracle_data_grib2.rs`
    - Oracle test **skips** when oracle files are missing (so CI/crates packaging is not blocked by large binary test data).

- Optional vendored reference sources (repo-only):
  - `vendor/` may contain third-party sources used for local reference and oracle comparisons (e.g. `libaec`).
  - `vendor/` is **excluded from crates.io packages** (see `Cargo.toml` `exclude`).

> Note: In the previous monorepo, there was additional Chinese documentation (e.g. `docs/rust-aec.md`). In a standalone repo, you can optionally move it here or condense it into README “Implementation notes”.

## 3) Public API (v0.1.0)

- `decode(input, params, output_samples) -> Result<Vec<u8>, AecError>`
- `decode_into(input, params, output_samples, output: &mut [u8]) -> Result<(), AecError>`
  - Improvement vs “always allocate a Vec”: callers can reuse a buffer to reduce allocations.
  - Still **one-shot**: requires `output_samples` ahead of time and writes a fixed-length output buffer.
- `flags_from_grib2_ccsds_flags(ccsds_flags: u8) -> AecFlags`
- Parameters:
  - `AecParams { bits_per_sample: u8, block_size: u32, rsi: u32, flags: AecFlags }`

## 4) Build & Verification Log (Windows)

Commands validated (run at crate root):

- Unit tests + (optional) oracle test + doctests:
  - `cargo test`
- Packaging check (crates.io dry-run):
  - `cargo package`

Key point: `cargo package` success indicates the publishable file set + metadata + README/License are acceptable for crates.io.

## 5) Oracle Test (how it works)

- Oracle idea:
  - Extract GRIB2 (template 42) **Section 7** AEC payload
  - Decode with **libaec** to generate oracle bytes
  - Decode with **rust-aec** and compare outputs **byte-for-byte**
- Current strategy:
  - `tests/oracle_data_grib2.rs` will **skip** when oracle files are missing → avoids CI failures and avoids shipping large binary data.
- Reproducing oracle data in a new repo:
  - Requires a small tool/script to extract payload and decode with libaec.
  - Recommendation: keep the oracle generator under an `xtask/` or an optional bin.
  - Do **not** make crates.io users depend on native libaec by default.