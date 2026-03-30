# Phase 1 Implementation Spec: Foundation & Utilities

## 1. Build System Foundation
**Goal**: Integrate Cargo into the existing Makefile and setup the Rust workspace.

- **Target Files**: `Makefile`, `rust/os-autoinst-core/Cargo.toml`, `rust/os-autoinst-core/src/lib.rs`.
- **Rust Setup**:
    - Use `pyo3` version `0.23` with `extension-module` feature.
    - Set `crate-type = ["cdylib", "rlib"]`.
    - `src/lib.rs` should include a `#[pymodule]` function named `os_autoinst_core` using the `Bound` API.
- **Makefile Update**:
    - Add `build-rust` target: `cd rust/os-autoinst-core && cargo build --release`.
    - Copy `target/release/libos_autoinst_core.so` to `rust/os-autoinst-core/os_autoinst_core.so` for easy Python import.
    - Hook `build-rust` into the `all` target.
- **Verification**: `make build-rust` succeeds.

## 2. Videoencoder (Rust)
**Goal**: Replace `videoencoder.cpp` with a Rust binary.

- **Target Files**: `rust/os-autoinst-core/src/bin/videoencoder.rs`.
- **Implementation**:
    - Spawn `ffmpeg` as a child process using `Stdio::piped()`.
    - Parameters: `-f image2pipe -vcodec ppm -r 24 -i - -vcodec libtheora -q:v 6 -y <output>`.
    - Handle `E <len>` (Emit frame) and `R` (Repeat last frame) commands from stdin.
    - Handle `live_log` symlinking to `qemuscreenshot/last.png`.
- **Verification**: Run a short test and verify `.ogv` presence.

## 2. Snd2png (Rust)
**Goal**: Replace `snd2png.cpp` with a Rust binary.

- **Target Files**: `rust/os-autoinst-core/src/bin/snd2png.rs`, `rust/os-autoinst-core/Cargo.toml`.
- **Dependencies**: `hound` (WAV parsing), `rustfft` (Spectrogram), `image` (PNG output).
- **Implementation**:
    - Replicate the windowing (10ms) and overlap logic from the original C++.
    - Use `rustfft` for the forward transformation.
    - Map magnitude to grayscale intensity.
- **Verification**: `make build-rust` and compare output PNG with legacy `snd2png`.
