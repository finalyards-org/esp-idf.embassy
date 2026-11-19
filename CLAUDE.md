# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A minimal Rust application for ESP32-C6 (`riscv32imac-esp-espidf`) combining ESP-IDF (`esp-idf-svc`, `std`) with the Embassy async executor. `src/main.rs` is intentionally tiny (logs a word per second); the code is not the point.

The repo has two purposes:
- **`main` branch**: a working getting-started template for ESP-IDF + Embassy.
- **`skinned` branch** (plus the `skinned/` folder and `Skinned.md`): a study of which ESP-IDF (C) components "seep in" uninvited, and which can be switched off. Findings are written up as Markdown reports in `skinned/` (`Report-6.1.md`, `BuildSize-6.1.md`, `BuildTimes-6.1.md`), targeting ESP-IDF 6.1. These reports are meant as a discussion starter with Espressif, not criticism.

## Build / run

```
cargo build --release -vv
cargo size --release         # flash/RAM sizes (cargo-binutils + llvm-tools); used in BuildSize reports
```

**Important agent rules for building & running:**
- **NEVER run `cargo clean`** unless explicitly instructed by the user. Full ESP-IDF rebuilds are very slow.
- **NEVER run `cargo run`** (or `espflash`). This environment is a headless VM without a physical ESP32-C6 devkit attached, so flashing/monitoring commands will hang or fail.
- **There are no tests** (`cargo test` is not applicable for this target).

The target dir is `~/target` rather than `./target` (use `cargo metadata` to find it).

The toolchain is a pinned nightly (`rust-toolchain.toml`) with `build-std`; don't switch to floating `nightly`.

## Where configuration lives

- `.cargo/config.toml` — target, `build-std`, linker/runner, and `[env]`: `MCU`, `ESP_IDF_VERSION` (switching versions = edit here + `cargo clean` + rebuild), and the (commented-out) `ESP_IDF_COMPONENTS` allow-list.
- `sdkconfig.defaults` — Kconfig overrides (stack sizes, deprecation-warning suppression). The `## EXPERIMENTS` section holds the skinning toggles being evaluated; the reports compare builds with that section on vs. commented out.
- `build.rs` — besides calling `embuild::espidf::sysenv::output()`, it deliberately **panics** if: run from RustRover (to avoid huge IDE-triggered ESP-IDF builds), `IDF_PATH` is set (a system/manual ESP-IDF shell is active), or a stray `sdkconfig` exists in the repo root. Build from a clean shell.

## `skinned/` folder workflow

`skinned/` is a dummy C ESP-IDF project (`CMakeLists.txt`, empty `main/main.c`) used only to run `idf.py menuconfig` against the `sdkconfig` that the Cargo build generated, to inspect why a Kconfig option is on (e.g. `ESP_WIFI_ENABLED` is forced `y` by `SOC_WIFI_SUPPORTED`). Do this inside `skinned/`, never the repo root (a root `sdkconfig` breaks the Rust build).

- `just sdkconfig-path` / `just project_description-path` (in `skinned/`, needs `jq`) — print the most recently generated `sdkconfig` / `project_description.json` under the Cargo target dir.
- Quick check without menuconfig: `grep KEY "$(just sdkconfig-path)"`.
- `idf.py` usage happens in a separate shell with ESP-IDF tooling exported (see `skinned/README.md`); exit that shell before running `cargo build`.

## Style

`rustfmt.toml`: max width 100, crate-granular vertical imports grouped Std/External/Crate, trailing commas.
