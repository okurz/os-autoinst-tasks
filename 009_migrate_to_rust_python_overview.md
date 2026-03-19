Spike: Evaluate Rust vs. Rust+Python architecture for os-autoinst

## Motivation
Modernize the `os-autoinst` backend. We already have a strong baseline with 100% statement coverage and a unified Makefile allowing us to decide on a new target architecture to evaluate: a 100% unified Rust codebase, or a hybrid approach using a high-performance Rust core (for video/VNC) orchestrated by a Python ecosystem (for API, backends, and test writing). A technical spike is needed to provide data for the final architectural decision.

## Acceptance criteria
* **AC1:** A technical evaluation document comparing pure Rust vs. Rust+Python (via PyO3) is published for the team.
* **AC2:** The evaluation includes a small proof-of-concept (PoC) demonstrating FFI overhead and developer ergonomics for both approaches.
* **AC3:** A decision matrix is presented at the team meeting, weighing performance, safety, and ease of contribution for external FOSS developers.
* **AC4:** No changes from this spike are merged into the production `os-autoinst` codebase.

## Suggestions
* Create a minimal dummy project to benchmark OpenCV needle matching in pure Rust vs. Rust called from Python via PyO3 (4-8h)
* Evaluate the build complexity of packaging a PyO3 bridge in the openSUSE OBS environment (2-4h)
* Draft the pros/cons matrix focusing on our common userbase and upstream contributors (1-2h)
* Optional: Draft a short example of what a Python-based `testapi` would look like for a test writer (1-2h)

## Further details
Keep in mind that while 100% Rust offers superior memory safety across the whole stack, Python significantly lowers the barrier to entry for external integration specialists and release teams writing custom backend extensions.

---

Phase 1: Migrate C++ Video/VNC core to Rust (Drop-in replacement)

## Motivation
The C++ components of `os-autoinst` (handling OpenCV image processing and VNC streams) are performance-critical but carry inherent memory-safety risks and complicate our build chain. Migrating this layer to Rust provides a modern, safe, and highly performant core. By designing it as a drop-in replacement, we can swap the engine while the existing Perl orchestration layer remains unaware of the change.

## Acceptance criteria
* **AC1:** C++ code for OpenCV needle matching and VNC handling is entirely replaced by Rust equivalents (e.g., using `opencv-rust`).
* **AC2:** The existing Perl orchestration logic can call the new Rust core via FFI without any behavioral changes.
* **AC3:** Existing 100% statement coverage invariants are maintained.
* **AC4:** All openQA-in-openQA tests pass successfully.
* **AC5:** The `Makefile` remains the primary entry point, seamlessly orchestrating `cargo` instead of `cmake` for the core build.

## Suggestions
* Add Rust/Cargo toolchain dependencies to the OBS package spec (1-2h)
* Create a Rust `cdylib` library exposing a C-compatible API for the Perl layer (2-4h)
* Port utility functions and serial console parsing to Rust and verify unit tests (4-8h)
* Port OpenCV needle matching logic to Rust and optimize (8-16h)
* Remove all legacy C++ code and `cmake` configurations (2-4h)

## Further details
Since we are interacting with Perl in this phase, `FFI::Platypus` or traditional XS will likely be required as the bridge. This bridge might be temporary depending on the outcome of the architecture spike (if we move to Python orchestration later).

---

Phase 2: Establish Orchestration Bridge and migrate peripheral backends

## Motivation
Assuming the decision to adopt a hybrid architecture (Rust Core + High-level scripting), this phase begins absorbing the Perl logic into the new ecosystem. By moving backend integrations (QEMU, Hyper-V, IPMI, etc.) and consoles to a modern scripting language (like Python), we make the codebase more accessible and extensible for platform engineers and external FOSS contributors.

## Acceptance criteria
* **AC1:** A native bridge (e.g., PyO3 for Python) is established, allowing the scripting layer to directly interface with the Phase 1 Rust core.
* **AC2:** At least one major backend module (e.g., QEMU execution and process management) is fully ported from Perl.
* **AC3:** The migrated backend interacts correctly with the Rust core (e.g., routing VNC streams).
* **AC4:** Other backend jobs (e.g., s390x, IPMI) are *not* affected and continue to run via the legacy Perl execution path until they are ported.
* **AC5:** 100% test coverage is proven for the newly migrated module.

## Suggestions
* Wrap the Phase 1 Rust core using the chosen bridge technology (e.g., PyO3) (4-8h)
* Port the internal data models (Job states, Configurations) from Perl hashes to strict representations (4-8h)
* Migrate the QEMU command-line generation and process spawning logic (8-12h)
* Implement a routing mechanism in the main daemon to direct test execution to either the legacy Perl backend or the new backend based on job configuration (4-8h)

## Further details
This phase uses the Strangler Fig pattern. We will run the newly ported backends in parallel or in staging environments (like `openqa.opensuse.org (o3)` test instances) to ensure there are no regressions in "Go/No-Go" signaling before rolling it out to production workers.
