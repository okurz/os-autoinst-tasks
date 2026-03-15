# Migration Plan: os-autoinst to Rust and Python

**Goal:** Migrate `os-autoinst` completely to Rust and Python, moving away from Perl and C/C++ dependencies. The only exception is maintaining a shallow Perl test-API to ensure existing test suites continue to function without modification. 

**Core Principles:**
- **Step-wise Migration:** Conducted in phases, keeping full functionality intact during every phase.
- **Backward Compatibility:** Keep backward-compatible wrappers to allow continuous operations while the migration is ongoing.
- **Quality:** Focus on fast, fully test-covered, human-readable, and sustainable code.

## Phase 1: Core Image Processing and VNC to Rust (The Foundation)
**Objective:** Replace the existing C++ and CMake controlled OpenCV/VNC component with a new, highly performant Rust core.

1. **Rust Core Implementation:**
   - Create a new Rust library (`os-autoinst-core`) to handle image processing (needle matching, OCR) and VNC communication.
   - Replicate the functionality of the existing C++ components with a focus on memory safety and concurrency.
   - Implement comprehensive unit tests in Rust for all image processing algorithms.
2. **Python Bridge (PyO3):**
   - Use `PyO3` to create a Python bridge to the `os-autoinst-core` Rust library.
   - Expose the necessary Rust structures and functions as a Python module (`os_autoinst_core`).
3. **Backward-Compatible Wrappers:**
   - Temporarily wrap the new Python module (or directly expose a C-API from Rust) to allow the existing Perl infrastructure to call into the new Rust core. This ensures the system remains fully functional while C++ code is deprecated.
4. **Validation:** Ensure all existing CTest and Perl test suites pass using the new Rust core.

## Phase 2: Python Migration of Main Executables (`isotovideo`)
**Objective:** Migrate the main execution flow (`isotovideo`) and core infrastructure from Perl to Python.

1. **Python `isotovideo` Rewrite:**
   - Port the core logic of `isotovideo` from Perl to Python.
   - Integrate the Python `isotovideo` with the `os_autoinst_core` Python module (from Phase 1).
2. **Shallow Perl Test-API:**
   - Refactor the existing Perl `testapi.pm` (and related modules) to act purely as a shallow interface.
   - These Perl modules will translate test API calls (e.g., `assert_screen`, `type_string`) into IPC calls or FFI calls to the new Python `isotovideo` backend.
3. **State Management and Event Loop:**
   - Implement the main event loop and state machine (running tests, handling timeouts) in Python (using `asyncio` if applicable).
4. **Validation:** Run the complete existing Perl test suite against the new Python-driven `isotovideo`.

## Phase 3: Backends and Consoles Migration
**Objective:** Port the backend management and console interactions to Python.

1. **Backend Migration (Python):**
   - Port `backend::base`, `backend::qemu`, `backend::svirt`, etc., to Python.
   - Ensure the new Python backends integrate seamlessly with the Python `isotovideo` loop.
2. **Console Migration (Python/Rust):**
   - Port console handling (`console::vnc`, `console::serial`, `console::ssh`) to Python.
   - Utilize the Rust core for high-performance VNC frame handling where necessary.
3. **Step-by-Step Integration:**
   - Migrate one backend/console at a time.
   - Use feature flags or configuration to switch between the legacy Perl implementation and the new Python implementation during testing.
4. **Validation:** Extensive testing with various backends (QEMU, bare-metal, Hyper-V) to ensure feature parity.

## Phase 4: Extensions, Utilities, and Cleanup
**Objective:** Migrate remaining components and remove deprecated code.

1. **Extensions Migration:**
   - Port any remaining Perl extensions or plugins to Python.
2. **Tooling and Utilities:**
   - Migrate supplementary scripts (e.g., in `tools/`) to Python.
3. **Code Deprecation and Removal:**
   - Once all phases are validated and running in production, systematically remove the old C++ and Perl implementations (excluding the shallow Perl test API).
   - Clean up the build system (remove CMake, adapt Makefiles to manage Rust/Cargo and Python dependencies).

## Quality Assurance & Testing Strategy
- **Continuous Integration:** Every PR must pass the full existing test suite. No regression in the Perl test API is permitted.
- **Coverage:** New Python and Rust code must have high test coverage (>90%).
- **Linting & Typing:** 
  - Python: Enforce strict type hints (`mypy`), linting (`ruff`), and formatting (`black`).
  - Rust: Enforce standard formatting (`rustfmt`) and linting (`clippy`).
- **Performance Benchmarks:** Establish benchmarks for needle matching and VNC frame processing to ensure the Rust implementation meets or exceeds the legacy C++ performance.
