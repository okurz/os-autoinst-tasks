# Phase 2: Python Migration of `isotovideo`

**Objective:** Port the main execution flow (`isotovideo`) and core infrastructure from Perl to Python.

## Step 1: Python `isotovideo` Scaffold
- Create `script/isotovideo.py` with basic argument parsing.
- Implement a basic `Runner` class in Python.
- Integrate with `os_autoinst_core` (the Rust bridge).

## Step 2: IPC and JSON-RPC
- Implement a JSON-RPC server/handler in Python to communicate with the Perl `autotest.pm`.
- Port `OpenQA::Isotovideo::CommandHandler` to Python.
- Support basic commands like `assert_screen` (calling into the Rust core).

## Step 3: Backend and Console Management
- Port `OpenQA::Isotovideo::Backend` and related classes to Python.
- Start managing QEMU/Svirt processes from Python.

## Step 4: Shallow Perl Test-API
- Refactor `testapi.pm` and `autotest.pm` to be thin wrappers that only send JSON-RPC requests to the Python `isotovideo`.

## Step 5: Validation
- Run existing Perl test suites using the new Python `isotovideo`.
- Ensure all logs and assets are correctly handled.
