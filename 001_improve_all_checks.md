# Plan to Optimize Runtime of All Checks

## Current State Analysis
- **Baseline Execution Time:** ~4 minutes 20 seconds (sequential `make test`).
- **Primary Bottleneck:** `test-perl-testsuite` takes ~232 seconds (sequential).
- **Secondary Bottleneck:** `test-local-perl-style` takes ~25 seconds.
- **Other Observations:** `test-local-bash-syntax` failed due to a formatting issue (fixed locally). Most other tests are sub-second or very fast.

## Proposed Optimizations

### 1. Parallelize Perl Test Suite (`test-perl-testsuite`)
The `tools/invoke-tests` wrapper calls `prove` without default parallelization.
- **Strategy:** Modify `tools/invoke-tests` to automatically detect the number of available processing units (e.g., via `nproc`) and set the `-j` (jobs) argument for `prove` if not already provided.
- **Goal:** Utilize all available CPU cores to drastically reduce the 232s runtime.
- **Estimated Improvement:** 50-70% reduction depending on core count and IO.

### 2. Parallelize Perl Style Checks (`test-local-perl-style`)
`tools/check-perl-style` invokes `perlcritic` on the root directory, causing it to check files sequentially.
- **Strategy:** Modify `tools/check-perl-style` to find relevant files and feed them to `perlcritic` using `xargs -P $(nproc)`.
- **Implementation Detail:** We may need to batch files (e.g., `xargs -n 20`) to balance process startup overhead vs. parallelism.
- **Goal:** Reduce the 25s runtime.

### 3. Parallelize Bash Script Checks (`test-local-bash-syntax`)
`tools/check-bash-scripts` currently pipes `git ls-files` to `xargs` without parallelism.
- **Strategy:** Add `-P $(nproc)` to the `xargs` invocation.
- **Goal:** Ensure this check scales well as the number of scripts increases, although it is currently fast.

### 4. General Improvements
- **Environment Variable:** Ensure `PROVE_ARGS` and other tools respect a common `NPROC` or similar variable if set, falling back to system detection.
- **Fixes:** Ensure `tools/check-bash-scripts` formatting is permanently fixed to prevent regressions in `test-local-bash-syntax`.

## Action Plan

1.  **Modify `tools/invoke-tests`**:
    - Add logic to detect CPU count.
    - Append `-j<count>` to `prove` arguments.
2.  **Modify `tools/check-perl-style`**:
    - Replace direct `perlcritic .` call with a `find ... | xargs -P ...` pipeline.
3.  **Modify `tools/check-bash-scripts`**:
    - Add `-P` to `xargs`.
4.  **Verification**:
    - Run `make test` to benchmark the new runtime.
    - Ensure `xt/27-make-update-deps.t` and other repo-consistency tests pass (commit local fixes if needed).
