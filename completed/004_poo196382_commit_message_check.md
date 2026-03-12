# Execution Plan: Conventional Commits Check with `gitlint`

This plan outlines the steps to implement a custom test rule that verifies if git commit messages follow the [Conventional Commits](https://www.conventionalcommits.org/) format using `python3-gitlint`.

## Objectives
- Integrate `gitlint` into the project's static analysis toolset.
- Enable the `contrib-title-conventional-commits` (CT1) rule.
- Ensure it runs during local developer checks (`make test-local`).
- Provide dedicated visibility for commit message checks in GitHub Pull Requests.
- Avoid redundant execution in CI by excluding it from the general "Author tests" job while keeping it in its own workflow.

## Steps

### 1. Update Dependencies
Add `python3-gitlint` to the project's dependency list to ensure it's available in the development container and CI environment.

- **File**: `dependencies.yaml`
- **Action**: Add `python3-gitlint` under `test_requires`.
- **Regenerate Files**: Run `make update-deps` to update `cpanfile`, `dist/rpm/os-autoinst.spec`, and `container/os-autoinst_dev/Dockerfile`.

### 2. Configure `gitlint`
Create a configuration file to enable the Conventional Commits rule and customize it if needed.

- **File**: `.gitlint` (Project Root)
- **Content**:
  ```ini
  [general]
  # Enable the conventional commits contrib rule
  contrib=contrib-title-conventional-commits

  [contrib-title-conventional-commits]
  # Optional: Customize types if they differ from the default
  # types=feat,fix,chore,docs,style,refactor,perf,test,revert,build,ci
  ```

### 3. Create Check Script
Create a helper script that handles both local execution and CI environments (checking PR ranges).

- **File**: `tools/check-git-commit-message`
- **Permissions**: Executable (`chmod +x`)
- **Content**:
  ```bash
  #!/bin/bash -e
  GITLINT="${1:-gitlint}"

  if [ -n "$GITHUB_BASE_REF" ]; then
      # In GitHub Actions, check the range from base branch to HEAD
      echo "Checking commit range: origin/${GITHUB_BASE_REF}..HEAD"
      # Ensure base branch is fetched
      git fetch origin "$GITHUB_BASE_REF" --depth=50
      "$GITLINT" --commits "origin/${GITHUB_BASE_REF}..HEAD"
  else
      # Local check: verify the current HEAD commit
      echo "Checking latest commit (HEAD)"
      "$GITLINT"
  fi
  ```

### 4. Integrate with Build System
Add the new test to the CMake configuration so it's included in `ctest` and `make test-local`.

- **File**: `cmake/test-targets.cmake`
- **Action**:
  - Add logic to find the `gitlint` executable.
  - Define `test-local-git-commit-message` using `add_test`.
- **Example Addition**:
  ```cmake
  find_program(GITLINT_PATH gitlint)
  if (GITLINT_PATH)
      add_test(
          NAME test-local-git-commit-message
          COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/tools/check-git-commit-message" "${GITLINT_PATH}"
          WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
      )
  else ()
      message(STATUS "Set GITLINT_PATH to the path of the gitlint executable to enable git commit message checks.")
  endif ()
  ```

### 5. Replace Current Workflow
Replace `.github/workflows/commit-message-checker.yml` with a new version that uses the integrated `gitlint` check.

- **File**: `.github/workflows/commit-message-checker.yml`
- **Content**:
  ```yaml
  ---
  name: 'Commit message check'
  # yamllint disable-line rule:truthy
  on:
    pull_request:
    push:
      branches:
        - '!master'

  jobs:
    check-commit-message:
      runs-on: ubuntu-latest
      container:
        image: registry.opensuse.org/devel/openqa/containers/os-autoinst_dev
      steps:
        - uses: actions/checkout@v4
          with:
            fetch-depth: 0
        - name: Run gitlint
          run: |
            git config --global --add safe.directory '*'
            make test-local TESTS="test-local-git-commit-message"
  ```

### 6. Adjust Author Tests (Avoid Redundancy)
Modify `.github/workflows/author-tests.yaml` to exclude the commit message check, as it now has its own dedicated workflow.

- **File**: `.github/workflows/author-tests.yaml`
- **Action**: Use `-E` (exclude) flag in `ctest` or `TESTS` exclusion pattern.
- **Example Modification**:
  ```yaml
  - name: make test-local
    run: |
      git config --global --add safe.directory '*'
      make
      # Run all local tests EXCEPT the commit message check
      ctest --output-on-failure -R "test-local-.*" -E "test-local-git-commit-message"
  ```

### 7. Verification
- **Local**:
  1. Install `gitlint` (e.g., `sudo zypper in python3-gitlint`).
  2. Run `make symlinks`.
  3. Run `make test-local` (verifies it's still included locally).
- **CI**:
  1. Commit and push the changes.
  2. Verify that the "Commit message check" workflow runs and succeeds/fails correctly.
  3. Verify that the "Author tests" workflow completes without running `test-local-git-commit-message`.
