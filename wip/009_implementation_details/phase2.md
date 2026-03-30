# Phase 2 Implementation Spec: Core Image Processing

## 1. Rust Core API
**Goal**: Implement the high-performance image processing functions in Rust.

- **Target Files**: `rust/os-autoinst-core/src/lib.rs`.
- **Dependencies**: `image`, `imageproc`.
- **Functions to Implement**:
    - `match_needle(screen, needle, x, y, w, h, margin)`: Use `match_template` with `SumOfSquaredErrors`. Must handle coordinate scaling and ROI.
    - `copy_rect(data, x, y, w, h)`: Return cropped image as PPM bytes.
    - `replace_rect(data, x, y, w, h, r, g, b)`: Return image with filled rectangle.
    - `scale(data, w, h)`: Return resized image (Lanczos3).
    - `save_image(data, path)`: Standard save to file.
    - `read_image(path)`: Load from file to PPM bytes.
    - `new_image(w, h)`: Create black image.
- **Verification**: Rust unit tests for each function.

## 2. Perl Bridge (Inline::Python)
**Goal**: Connect Perl to the Rust core using `Inline::Python`.

- **Target Files**: `ppmclibs/tinycv.pm`.
- **Implementation**:
    - `_init_rust_bridge()`: Load `Inline::Python`, setup `sys.path` to include `PROJECT_ROOT/rust/os-autoinst-core`.
    - Import `os_autoinst_core`.
    - Handle initialization errors gracefully with `bmwqemu::fctwarn`.
- **Verification**: `perl -e 'use tinycv; tinycv::_init_rust_bridge()'` succeeds.

## 3. Pure Perl `tinycv`
**Goal**: Replace XS/C++ backend with the Rust-Python bridge.

- **Target Files**: `ppmclibs/tinycv.pm`.
- **Implementation**:
    - Define `package tinycv::Image` using `Mojo::Base -base`.
    - Member: `ppm_data` (scalar containing raw bytes).
    - Methods:
        - `search_needle`: Call `os_autoinst_core.match_needle` via `py_call_function`.
        - `write`: Call `os_autoinst_core.save_image`.
        - `copyrect`, `replacerect`, `scale`: Return new `tinycv::Image` instances.
        - `xres`, `yres`: Parse PPM header `P6\nWIDTH HEIGHT\n`.
- **Verification**: `prove t/01-test_needle.t` with `OS_AUTOINST_RUST_CORE=1`.

## 4. Debugviewer (Python)
**Goal**: Replace `debugviewer.cpp` with a Python version.

- **Target Files**: `script/debugviewer.py`, `Makefile`.
- **Implementation**:
    - Use `tkinter` for the GUI and `Pillow` (PIL) for image handling.
    - Logic: Loop every 300ms, check if the file exists, load it, and display it.
    - Bind `<Escape>` to exit.
- **Makefile Update**: Add symlink `debugviewer/debugviewer -> ../script/debugviewer.py`.
- **Verification**: Run `debugviewer/debugviewer some_image.png` manually.
