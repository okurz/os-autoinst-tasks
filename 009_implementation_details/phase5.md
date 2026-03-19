# Phase 5 Implementation Spec: Cleanup & Finalization

## 1. Comprehensive Validation
**Goal**: Ensure parity with legacy stack.

- **Tasks**:
    - Run full test suite in `local/no_vars`.
    - Verify logs, assets, and JSON outputs are identical to legacy.
    - Check performance (RAM/CPU) of the new Rust core.
- **Verification**: `make check` and manual review.

## 2. Dependency Purge
**Goal**: Remove C++ dependencies from metadata.

- **Target Files**: `dependencies.yaml`, `cpanfile`, `dist/rpm/os-autoinst.spec`.
- **Implementation**:
    - Remove `gcc-c++`, `pkgconfig(opencv4)`, `pkgconfig(fftw3)`, `pkgconfig(sndfile)`, `cmake`, `ninja`.
    - Add `python3-Pillow`, `python3-tkinter` (for debugviewer).
- **Verification**: `make update-deps` succeeds and `xt/27-make-update-deps.t` passes.

## 3. C++/CMake Deletion
**Goal**: Physical removal of legacy code.

- **Target Files**: `CMakeLists.txt`, `videoencoder.cpp`, `snd2png/*.cpp`, `debugviewer/*.cpp`, `ppmclibs/*.cc`, `ppmclibs/*.xs`.
- **Implementation**: `git rm` all listed files and sub-directories.
- **Verification**: `find . -name "*.cpp"` returns nothing.
