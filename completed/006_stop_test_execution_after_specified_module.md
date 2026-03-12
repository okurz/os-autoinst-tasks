# Task 6: Stop test execution after a specified test module

Add test coverage and implementation to stop test execution after a specified test module, similar to `INCLUDE_MODULES` and `EXCLUDE_MODULES`. This is based on the WIP commit `4fb03de5`.

## Problem Description

Currently, openQA/os-autoinst allows including or excluding specific test modules using `INCLUDE_MODULES` and `EXCLUDE_MODULES`. However, there's no easy way to say "run everything up to this module and then stop". A new variable `EXIT_AFTER` should be implemented to support this.

## Proposed Changes

### `basetest.pm`

- Update `is_applicable` to support `EXIT_AFTER`.
- The logic should check if any module already added to the schedule (`@autotest::testorder`) matches the name or fullname specified in `$bmwqemu::vars{EXIT_AFTER}`.
- If a match is found, subsequent modules should return `false` for `is_applicable`.

### `t/17-basetest.t`

- Add unit tests to verify the `EXIT_AFTER` logic in `is_applicable`.
- Test cases should include:
    - `EXIT_AFTER` matching a module name.
    - `EXIT_AFTER` matching a module fullname.
    - Interaction with `INCLUDE_MODULES` and `EXCLUDE_MODULES`.
    - Verification that the module matching `EXIT_AFTER` is still applicable itself, but subsequent ones are not.

## Implementation Details

The implementation in `basetest::is_applicable` will look like this:

```perl
sub is_applicable ($self) {
    # ... existing EXCLUDE_MODULES and INCLUDE_MODULES logic ...

    if (my $exit_after = $bmwqemu::vars{EXIT_AFTER}) {
        # If any previously scheduled module matched EXIT_AFTER, this one is not applicable
        return 0 if grep { $_->{class} eq $exit_after || $_->{fullname} eq $exit_after } @autotest::testorder;
    }
    return 1;
}
```

This approach is safe because:
1. It doesn't require additional global state; it uses the existing `@autotest::testorder`.
2. It correctly allows the matching module to be scheduled (as it's not yet in `@testorder` when `is_applicable` is called for it).
3. It naturally stops further scheduling.

## Verification Plan

### Automated Tests
- Run `make test-perl-testsuite TESTS="t/17-basetest.t"` to verify the new unit tests.
- Run `make check` to ensure no regressions in other parts of the system.

### Manual Verification
- Can be verified by running `isotovideo` with a dummy test suite and setting `EXIT_AFTER`.
- Example: `EXIT_AFTER=boot isotovideo ...` should stop after the `boot` module is scheduled/run.

## Documentation
- The variable `EXIT_AFTER` is already documented in `doc/backend_vars.asciidoc` in the WIP commit.
- Ensure the behavior matches the documentation: "name or fullname of test module after which the schedule will stop".
