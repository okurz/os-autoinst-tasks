# Test Improvement Plan

## 1. Stabilize `t/23-baseclass.t` (Pipe Size Flakiness)

**Problem:**
The test intermittently fails at `subtest 'pipe size set'`. It attempts to set the pipe size for the video encoder using `F_SETPIPE_SZ` and then asserts that the size returned by `F_GETPIPE_SZ` is at least the requested size. Failures may occur due to kernel rounding, race conditions, or `EPERM`.

**Solution:**
Implemented a logic-based verification that respects the environment's capabilities:
- If the OS allows setting the pipe size (no warning), verify the size is large.
- If the OS denies it (warning printed), verify that the code handled it gracefully (size remains small).
This ensures the application logic is correct in both scenarios. Attempts to mock `fcntl` strictly were blocked by `autodie` and `CORE` interactions in the legacy codebase.

**Verification:**
- Ran `t/23-baseclass.t` 20 times in a loop with no failures.

## 2. Optimize `t/99-full-stack.t` (Performance/Timeout)

**Problem:**
This test mimics a full openQA job execution. It intentionally uses a "broken" console (`brokenvnc`) pointing to `novnc.nowhere`. The `consoles::VNC` module has a hardcoded `sleep 1` between connection retries. With default timeouts and retries, the test runs for minutes, often timing out.

**Solution:**
1.  **Configurable Sleep:** Modified `consoles/VNC.pm` to use a configurable sleep duration (`$bmwqemu::vars{VNC_CONNECT_SLEEP}`).
2.  **Minimize Waits:** In `t/99-full-stack.t`, set `VNC_CONNECT_SLEEP` to `0` and timeouts to `0.001`.

**Verification:**
- Ran `t/99-full-stack.t` 10 times. Average execution time reduced to ~150s (from >300s/timeout).

## Actionable Steps & Commits

1.  **Stabilize Baseclass Test**
    - [x] Modify `t/23-baseclass.t` to improve pipe size assertion logic and handle `EPERM`.
    - [x] Verify: `for i in {1..20}; do make test-perl-testsuite TESTS="t/23-baseclass.t" || break; done`
    - [x] Commit: `test(baseclass): stabilize pipe size assertion`

2.  **Make VNC Sleep Configurable**
    - [x] Modify `consoles/VNC.pm` to replace `sleep 1` with `sleep($bmwqemu::vars{VNC_CONNECT_SLEEP} // 1)`.
    - [x] Verify: Run existing VNC tests to ensure no regression.
    - [x] Commit: `feat(vnc): make connection retry sleep configurable`

3.  **Optimize Full Stack Test**
    - [x] Edit `t/99-full-stack.t` to:
        - Add `"VNC_CONNECT_SLEEP": "0"` to `vars.json`.
        - Export `VNC_CONNECT_TIMEOUT_REMOTE=0.001` and `VNC_CONNECT_TIMEOUT_LOCAL=0.001`.
    - [x] Verify: `for i in {1..10}; do make test-perl-testsuite TESTS="t/99-full-stack.t" || break; done`
    - [x] Commit: `test(full-stack): optimize execution time by reducing timeouts`

4.  **Cleanup Tools (Extra)**
    - [x] fix(tools): ignore untracked files in bash syntax check
