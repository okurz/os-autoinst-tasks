# Proposals to Improve CI Resiliency for Image Pulls

This document outlines three proposals to address the recurring failure of Docker image pulls from `registry.opensuse.org` in GitHub Actions.

## Proposal 1: Mirror Images to GitHub Container Registry (GHCR) (Recommended)

Since GitHub Actions runners are hosted within GitHub's infrastructure, network issues between runners and GHCR (`ghcr.io`) are almost non-existent.

### Implementation
1.  **Mirroring Workflow:** Create a dedicated GitHub Action (`.github/workflows/mirror-images.yml`) that runs on a schedule (e.g., daily) and on-demand to pull the latest images from `registry.opensuse.org/devel/openqa/containers/os-autoinst_dev` and push them to `ghcr.io/os-autoinst/os-autoinst_dev`.
2.  **Workflow Update:** Update existing CI files (`.github/workflows/*.yml`) to use the new image path.

### Pros
-   **High Reliability:** Native to GitHub infrastructure.
-   **No Architectural Changes:** Keeps the existing `container:` block structure in workflows.
-   **Simplified Permissions:** Pulling from a public GHCR image is often more stable than external registries.

### Cons
-   **One-time Setup:** Requires a one-time configuration of a push token with package write permissions.

---

## Proposal 2: Manual Container Execution with Failover

Instead of using the native `container:` job block, we move container execution into manual steps within the job.

### Implementation
1.  **Manual Pull Step:** Use a step that attempts to pull from a primary registry (e.g., `registry.opensuse.org`) with a loop or fallback to a secondary registry (e.g., a community-maintained mirror on `docker.io` or `ghcr.io`).
2.  **Manual Run Step:** Execute commands via `docker run --workdir /opt -v "$PWD":/opt ...`.

### Pros
-   **Maximum Flexibility:** Allows for custom retry logic and failover to multiple registries.
-   **Fine-grained Control:** Can handle complex authentication or networking scenarios.

### Cons
-   **Increased YAML Verbosity:** Each job needs setup steps for volume mounting and environment variable passing.
-   **Complexity:** Harder to maintain across multiple workflow files.

---

## Proposal 3: Image Layer Caching via GitHub Actions Cache

Leverage GitHub's internal caching mechanism to store the Docker image layers between runs.

### Implementation
1.  **Cache Step:** Use `actions/cache` or specialized actions (like `satackey/action-docker-layer-caching`) to save the pulled Docker layers.
2.  **Workflow Logic:** If a cache hit occurs, the `docker pull` will only need to check for updates rather than downloading all layers.

### Pros
-   **Speed:** Significant performance boost on cache hits.
-   **Reduced External Dependency:** Registry is only contacted if the cache is empty or the image is updated.

### Cons
-   **Storage Limits:** Consumes part of the 10GB per-repository cache limit.
-   **Incompatibility:** Not easily compatible with the native `container:` block, as GitHub pulls that image *before* any steps (including cache restoration) run. This would require moving to manual `docker run` as well.
