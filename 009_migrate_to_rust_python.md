# Revised Migration Plan: os-autoinst to Rust and Python

**Goal:** Completely migrate `os-autoinst` to Rust and Python, removing all C/C++ and CMake dependencies while preserving a shallow Perl test-API for existing suites. This plan incorporates learnings from initial prototyping to ensure a robust, maintainable, and verifiable transition.

---

## Phase 1: Foundation & Low-Risk Utilities (Trust Building)
*Objective: Establish the new stack and replace non-critical C++ tools first.*

1.  **Build System Foundation**:
    - Integrate `cargo` into the top-level `Makefile`.
    - Setup `rust/os-autoinst-core` with `PyO3` support.
2.  **Debugviewer (Python)**:
    - Replace the legacy C++ `debugviewer` with a Python implementation using `Tkinter` and `Pillow`.
    - *Verification*: Manual check of image auto-refresh functionality.
3.  **Snd2png (Rust)**:
    - Rewrite the audio spectrogram generator in Rust using `rustfft` and `hound`.
    - *Verification*: Compare output PNGs with legacy `snd2png` outputs.
4.  **Videoencoder (Rust)**:
    - Rewrite the video encoding binary in Rust, managing an `ffmpeg` pipe for Ogg Theora.
    - *Verification*: Verify `.ogv` recording in standard test runs.

---

## Phase 2: Core Image Processing (Rust Core)
*Objective: Replace the performance-critical `tinycv` C++ logic with Rust.*

1.  **Rust Core API**:
    - Implement `match_needle` (SSE-based, ROI/margin support), `copyrect`, `replacerect`, and `scale` in the Rust core.
2.  **Perl Bridge (Inline::Python)**:
    - Utilize the existing `Inline::Python` dependency to call into the Rust core (via PyO3). 
    - *Decision*: Avoid `FFI::Platypus` due to availability constraints in openSUSE.
3.  **Pure Perl `tinycv`**:
    - Refactor `ppmclibs/tinycv.pm` to be a pure Perl bridge, removing all XS and C++ interdependencies.
4.  **Verification**:
    - Run existing `t/01-test_needle.t` with `OS_AUTOINST_RUST_CORE=1`.

---

## Phase 3: Modern Orchestrator (Python isotovideo)
*Objective: Port the main execution loop and IPC to Python.*

1.  **Orchestrator Scaffold**:
    - Implement `isotovideo.py` with robust absolute path resolution and `PROJECT_ROOT` discovery.
2.  **Hardened IPC Bridge**:
    - Implement `JsonRpcStream` to handle stream buffering and message extraction correctly (avoiding brittle line-splitting).
3.  **Command Dispatcher**:
    - Implement a clean Registry/Dispatcher pattern for command handling to avoid monolithic `if/elif` blocks.
4.  **Verification**:
    - Integration test `t/98-python-isotovideo.t` ensuring IPC reliability.

---

## Phase 4: Backend & Console Protocols (Python)
*Objective: Port VM management and console communication.*

1.  **QEMU Infrastructure**:
    - Port `QemuBuilder` (command-line generation) and `QmpClient` (Machine Protocol).
2.  **VNC Protocol**:
    - Implement RFB handshake, keysym mapping, and `ikvm` (USB) support in Python.
3.  **Dynamic Console Registry**:
    - Restore multi-console support via lazy instantiation in the Python `Runner`.
4.  **OCR Bridge**:
    - Migrate Tesseract integration with multilingual support to the Python backend.

---

## Phase 5: Cleanup & Finalization
*Objective: Remove all legacy debt and verify full system operation.*

1.  **Comprehensive Validation**:
    - Conduct full test suite validation running entirely on the new stack (`local/no_vars` and full `openQA` scenarios).
2.  **Dependency Purge**:
    - Remove `gcc-c++`, `OpenCV`, `fftw`, `cmake`, and `ninja` from `dependencies.yaml`.
3.  **C++/CMake Deletion**:
    - Systematically remove all `.cpp`, `.cc`, `.h`, `.xs`, and `CMakeLists.txt` files.

---

## Maintainability & Implementation Principles
- **Commit Granularity**: Every numbered step must be a single, atomic, and fully verified commit.
- **Zero Duplication**: Shared logic (e.g., image parsing) must be centralized in the Rust core.
- **Incremental Switch**: Use the `OS_AUTOINST_RUST_CORE=1` environment variable to allow side-by-side testing of the legacy and new stacks.
- **Human Readable**: Prioritize functional style and clear naming over procedural complexity.
