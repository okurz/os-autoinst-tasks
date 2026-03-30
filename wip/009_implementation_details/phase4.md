# Phase 4 Implementation Spec: Backend & Console Protocols

## 1. QEMU Infrastructure
**Goal**: Manage QEMU from Python.

- **Target Files**: `python/os_autoinst/qemu_builder.py`, `python/os_autoinst/qmp.py`, `python/os_autoinst/backend.py`.
- **Implementation**:
    - `QemuBuilder`:
        - Replicate parameter mapping for `-m`, `-smp`, `-cpu`, `-drive`, `-netdev`, `-vnc`.
        - Handle `VNC_EXTRA_VARS`.
    - `QmpClient`:
        - Connect to `qmp.sock`.
        - Handshake: `qmp_capabilities`.
        - Support `quit` and `cont` commands.
- **Verification**: `python3 t/python/test_qemu_builder.py`.

## 2. VNC Protocol
**Goal**: Native Python VNC client for input.

- **Target Files**: `python/os_autoinst/console.py`.
- **Implementation**:
    - `VNCConsole` class:
        - Socket-based RFB handshake (`003.008`).
        - Security negotiation (Support 'None' type).
        - PixelFormat negotiation (32bpp, 24depth, little-endian).
        - Encodings (Raw).
        - `send_key(keycode, down)`: Use `struct.pack(">BBHI", 4, ...)` for KeyEvent.
        - Handle `ikvm` padding (`BxBHI9x`).
- **Verification**: `t/97-python-vnc-console.t` with mock VNC server.

## 3. Dynamic Console Registry
**Goal**: Restore multi-console support.

- **Target Files**: `script/isotovideo.py`.
- **Implementation**:
    - `Runner` keeps `self.consoles` dict.
    - `select_console(name)`:
        - If not in dict, create instance (e.g. `VNCConsole`).
        - If exists, call `activate()`.
- **Verification**: Integration test switching between VNC and Serial.

## 4. OCR Bridge
**Goal**: Migrate Tesseract port to Python.

- **Target Files**: `script/isotovideo.py`, `ocr.pm`.
- **Implementation**:
    - Python: Receive `ocr` command with Base64 PPM data.
    - Save to temp PNG, run `tesseract <tmp> ocr_out -l <lang>`.
    - Return text from `ocr_out.txt`.
- **Verification**: `t/96-python-ocr.t`.
