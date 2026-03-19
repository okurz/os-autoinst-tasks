# Execution Plan: Implement `always_run` test flag

## Objective
Extend `test_flags` with an `always_run` flag to ensure that a test module executes even if a previous test module encountered a fatal failure. This is especially useful for cleanup tasks (e.g., freeing remote testing resources).

## Files to Modify

### 1. `basetest.pm`
*   **Documentation:** Update the POD documentation for `test_flags` (around line 90) to include the new flag:
    ```perl
      'always_run'      - requests that a test module is always executed regardless if a previous test module is set to 'fatal'
    ```

### 2. `autotest.pm`
*   **Modify `runalltests` to defer stopping the VM and returning `0`:**
    *   Initialize a state variable `my $fatal_reason;` before the main `for` loop over `@testorder`.
    *   **Skip logic update:** Before running a test, check if a fatal error already occurred. Skip the test unless it has the `always_run` flag.
        ```perl
        if (!$vmloaded) {
            bmwqemu::diag "skipping $fullname";
            $t->skip_if_not_running();
            $t->save_test_result();
            next;
        }
        if ($fatal_reason && !$flags->{always_run}) {
            bmwqemu::diag "skipping $fullname (after fatal failure)";
            $t->skip_if_not_running();
            $t->save_test_result();
            next;
        }
        ```
    *   **Fatal failure processing:** Inside the `if ($error)` block, replace the immediate `bmwqemu::stop_vm()` and `return 0` logic with setting `$fatal_reason`. This ensures the loop continues so `always_run` tests have a chance to execute.
        ```perl
        if ($t->{fatal_failure} || $flags->{fatal} || (!exists $flags->{fatal} && !$snapshots_supported) || $bmwqemu::vars{TESTDEBUG}) {
            $fatal_reason //= ($t->{fatal_failure} || $flags->{fatal})
              ? 'after a fatal test failure'
              : ($bmwqemu::vars{TESTDEBUG}
                ? 'because TESTDEBUG has been set'
                : 'because snapshotting is disabled/unavailable and "fatal => 0" has NOT been set explicitly');
            bmwqemu::diag "scheduled stop of overall test execution $fatal_reason";
        }
        elsif (defined $next_test && !$flags->{no_rollback} && $last_milestone && !$fatal_reason) {
            load_snapshot('lastgood');
            $next_test->record_resultfile('Snapshot', "Loaded snapshot because '$name' failed", result => 'ok');
            rollback_activated_consoles();
        }
        ```
        *(Notice `!$fatal_reason` was added to the `elsif` condition so we do not attempt a rollback after encountering a fatal failure).*

    *   **Success path constraints:** Wrap the snapshot operations and rollback logic in the `else` block (when no error occurs) inside an `if (!$fatal_reason)` condition. This prevents `always_run` tests that successfully execute from mistakenly generating or reverting snapshots when the overall state is already fatal.
    *   **Cleanup and Exit:** After the test loop completes, if `$fatal_reason` was set, handle the deferred teardown:
        ```perl
        if ($fatal_reason) {
            bmwqemu::diag "stopping overall test execution $fatal_reason";
            bmwqemu::stop_vm();
            return 0;
        }
        return 1;
        ```

### 3. `t/08-autotest.t`
*   **Add Unit Tests:**
    *   Add a new `subtest` for the `always_run` flag.
    *   Mock up a test schedule with three test modules:
        1. A test module with `fatal => 1` that throws an error/fails.
        2. A normal test module.
        3. A test module with `always_run => 1` that runs successfully.
    *   Verify that `autotest::run_all` skips the second module and executes the third module.
    *   Assert that `run_all` eventually ends with `completed` = 0 (since a fatal failure did occur) and `died` = 0.
    *   Assert that no snapshots are loaded (`load_snapshot`) or made (`make_snapshot`) after the initial fatal failure.
