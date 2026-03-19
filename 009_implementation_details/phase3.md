# Phase 3 Implementation Spec: Modern Orchestrator

## 1. Orchestrator Scaffold
**Goal**: Create the Python `isotovideo` script.

- **Target Files**: `script/isotovideo.py`.
- **Implementation**:
    - Handle `--debug`, `--workdir`, and `--version` arguments.
    - Implement dynamic `PROJECT_ROOT` detection:
      ```python
      script_dir = os.path.dirname(os.path.abspath(__file__))
      project_root = script_dir if os.path.exists(os.path.join(script_dir, "python")) else os.path.abspath(os.path.join(script_dir, ".."))
      ```
    - Implement `Runner.start_autotest()`:
      - Spawn Perl process: `perl -e 'use lib "..."; use autotest; ... runalltests();'`.
      - Ensure `autotest` is connected to the Python JSON-RPC socket.
- **Verification**: `./isotovideo.py --version` shows correct interface version.

## 2. Hardened IPC Bridge
**Goal**: Reliable JSON-RPC communication.

- **Target Files**: `script/isotovideo.py`.
- **Implementation**:
    - `JsonRpcStream` class:
        - `self.buffer`: Accumulate raw bytes from `recv`.
        - `feed(data)`: Split by `\n`, parse each line as JSON.
        - Handle `UnicodeDecodeError` or partial JSON gracefully.
    - Use `select.select` for non-blocking I/O on the Unix socket.
- **Verification**: `t/98-python-isotovideo-integration.t` (Perl client sending multiple commands).

## 3. Command Dispatcher
**Goal**: Clean routing of JSON-RPC commands.

- **Target Files**: `script/isotovideo.py`.
- **Implementation**:
    - `CommandHandler` class:
        - `self.methods`: Map string command names to Python methods.
        - Supported commands: `match_needle`, `select_console`, `set_current_test`, `read_serial`, `ocr`, `send_key`.
        - Routing: `backend_*` commands forwarded to `runner.backend.handle_command`.
- **Verification**: Integration test verifying successful dispatch of `backend_can_handle`.
