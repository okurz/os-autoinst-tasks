# Plan: Better Python Checks and Tests

## Objective
Adopt the high-quality Python code checks, formatting, and test standards from the `qem-bot` reference repository and apply them to `os-autoinst`. This will establish strong guardrails for the planned expansion of Python content in this repository.

## Phase 1: Configuration (`pyproject.toml`)
We will centralize our Python tooling configuration in `pyproject.toml`, adopting `qem-bot`'s best practices.

1. **Replace Black with Ruff**:
   - Remove the existing `[tool.black]` section.
   - Copy `qem-bot`'s `[tool.ruff]` configurations, including `line-length = 120`.
   - Copy the extensive `[tool.ruff.lint]` rules (`extend-select` with `E`, `F`, `D`, `Q`, `PL`, `S`, etc.) and `ignore` lists.
   - Adapt `[tool.ruff.lint.per-file-ignores]` to target `os-autoinst`'s test directory layout (e.g., `t/**/*.py` instead of `tests/*`).
   - Copy `[tool.ruff.format]` and `[tool.ruff.lint.flake8-tidy-imports.banned-api]` settings.

2. **Testing & Coverage Configuration**:
   - Add `[tool.pytest.ini_options]` configured to run out of `t/` (or `t/fake/tests/` as applicable).
   - Add `[tool.coverage.run]` and `[tool.coverage.report]` sections (with `fail_under = 100` or a more realistic initial threshold for existing code, gradually increasing).

3. **Type Checking (Ty)**:
   - Add `[tool.ty.terminal]` configuration (`error-on-warning = true`).

4. **Dependencies**:
   - Add required Python packages (`ruff`, `vulture`, `radon`, `ty`, `pytest`, `pytest-cov`, `pytest-mock`, `pytest-xdist`) to `dependencies.yaml` (since `os-autoinst` uses this file for tracking system dependencies) and potentially as a `dev` dependency group in `pyproject.toml`.

## Phase 2: Build System Integration (`CMakeLists.txt` / `Makefile`)
Since `os-autoinst` primarily uses CMake (with `Makefile` as a wrapper), we will expose the python checks as CMake custom targets, akin to the existing `tidy-cpp` and test targets. 

Add the following targets (likely in `cmake/test-targets.cmake` or `CMakeLists.txt`):

1. `check-python-style`: Runs `ruff check` and `ruff format --check`.
2. `tidy-python`: Auto-formats and fixes issues via `ruff format` and `ruff check --fix`.
3. `check-python-code-health`: Runs `vulture $(find . -name "*.py") --min-confidence 80`.
4. `check-python-maintainability`: Runs `radon mi $(find . -name "*.py") -n B`.
5. `check-python-conventions`: Custom bash check for `@patch` (as used in `qem-bot` to prevent argument ordering bugs).
6. `typecheck-python`: Runs `ty check`.
7. `test-python`: Runs dynamic tests with coverage using `python3 -m pytest -n auto -v --cov --cov-report=xml --cov-report=term-missing`.
8. `check-python`: Aggregate target depending on all static checks.

*Note: Ensure these integrate seamlessly with the existing `make check` target so they are executed in CI.*

## Phase 3: Applying Checks to Existing Code
The current footprint of Python is small but needs to be brought up to standard before the rules can be enforced continuously:

1. Run `make tidy-python` (or `ninja tidy-python`) to automatically reformat `script/crop.py`, `os_autoinst_setup.py`, and the Python tests in `t/`.
2. Run `vulture` and address any legitimate dead code.
3. Run `radon` and refactor any complex code failing the maintainability index.
4. Run `ty check` and add necessary type hints to existing Python code.
5. Fix any residual `ruff` warnings manually.

## Phase 4: CI & Hooks integration
1. Ensure the overarching `check` target or GitHub Actions workflow invokes the newly created `check-python` and `test-python` targets.
2. (Optional) Provide an updated `make setup-hooks` to ensure `pre-commit` encompasses Python checks if developers prefer it locally.
