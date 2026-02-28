# Plan for Optimizing Test Runtime

## Overview
Based on initial measurements using `prove -I. -Ilib --timer -j4 t/`, several tests have been identified as significant contributors to the total test suite runtime. This plan outlines strategies to reduce their execution time.

## Identified Slow Tests
| Test File | Runtime (ms) | Category |
|-----------|--------------|----------|
| `t/99-full-stack.t` | ~148,000 | Integration (QEMU) |
| `t/00-compile-check-all.t` | ~38,000 | Static Analysis |
| `t/14-isotovideo.t` | ~10,000 | Integration |
| `t/18-qemu-options.t` | ~3,700 | Integration |

## Root Cause Analysis
1. **Full Process Overhead**: many tests (e.g., `t/14-isotovideo.t`, `t/18-qemu-options.t`, `t/99-full-stack.t`) execute `isotovideo` via `system()` or backticks. This incurs the overhead of Perl interpreter startup, module loading, and full system initialization for each subtest.
2. **Monolithic Static Analysis**: `t/00-compile-check-all.t` checks every Perl file in the repository sequentially. As the codebase grows, this test will continue to slow down.
3. **Hardware Emulation**: `t/99-full-stack.t` and some others actually spawn QEMU, which is inherently slow compared to pure software mocks.

## Optimization Strategies

### 1. Parallelization of Monolithic Tests
* **Action**: Split `t/00-compile-check-all.t` into multiple smaller tests or use a parallel-capable checker.
* **Benefit**: Allows `prove -j` to distribute the load across multiple CPU cores.
* **Implementation**: Create a script to generate multiple `.t` files for different subdirectories (e.g., `t/00-compile-check-backend.t`, `t/00-compile-check-lib.t`).

### 2. Reducing `isotovideo` Spawning
* **Action**: Refactor tests that use `system(".../isotovideo ...")` to use internal method calls or a shared, pre-loaded environment where possible.
* **Benefit**: Eliminates the overhead of repeatedly starting a new Perl process and loading all dependencies.
* **Implementation**: Use `Test::MockModule` or similar to mock at a higher level, or create a "lite" version of `isotovideo` for testing that skips heavy initialization.

### 3. Improving `t/99-full-stack.t`
* **Action**: Verify if all assertions in `t/99-full-stack.t` require a full QEMU run.
* **Benefit**: Potentially reduces the number of QEMU starts.
* **Implementation**: If some checks are about log parsing, they might be moved to tests that use a mocked backend.

### 4. Caching for Static Analysis
* **Action**: Implement a caching mechanism for `t/00-compile-check-all.t`.
* **Benefit**: Skips checking files that haven't changed since the last successful run.
* **Implementation**: Use a simple hash-based cache (e.g., storing MD5 of files and their check status).

## Next Steps
1. **Experiment with splitting `t/00-compile-check-all.t`** and measure the gain with `prove -j8`.
2. **Profile `t/99-full-stack.t`** to see where exactly the time is spent (QEMU startup vs. test execution).
3. **Investigate `Test::ParallelSubtest`** for tests with many subtests like `t/14-isotovideo.t`.
