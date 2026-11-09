# CLM
Also called: CPU-GPU Memory Barrier Removal, Smart Caching Policy for Gaussian States

## Overview
CLM is an optimization technique for 3D Gaussian Splatting that eliminates the performance bottleneck of GPU memory barriers by dynamically migrating Gaussian states between CPU and GPU memory [S1, unofficial]. It solves the problem of excessive data transfer overhead during training by employing a smart caching policy that retains frequently updated Gaussians on the GPU while offloading less active ones to CPU RAM [S1, unofficial]. This approach differs from static partitioning methods like Grendel-GS [S1, unofficial] and out-of-core streaming frameworks like TideGS [S1, unofficial] by focusing on temporal locality of updates rather than spatial partitioning or disk-based streaming.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussian parameters $\theta = \{\mathbf{p}_i, \mathbf{\Sigma}_i, \alpha_i, \mathbf{c}_i, \mathbf{S}_i\}$ (position, covariance, opacity, color, spherical harmonics coefficients) distributed across CPU and GPU memory.
*   **Output:** Updated Gaussian parameters after an optimization step, with the subset of "hot" Gaussians residing in GPU memory for immediate rasterization.

**Data Structures:**
*   **Gaussian State:** A struct containing all attributes of a single Gaussian primitive.
*   **Cache Set:** A fixed-size buffer in GPU memory holding the most recently accessed or updated Gaussians.
*   **Liveness Counter/Flag:** A mechanism (e.g., a timestamp or usage counter) associated with each Gaussian to determine its "hotness" [G].

**Steps:**
1.  **Initialization:** Load the initial set of Gaussians into CPU RAM. [S1, unofficial]
2.  **Selection:** Identify the subset of Gaussians required for the current training view. This typically includes Gaussians visible in the current camera frustum and those recently updated in previous iterations [S1, unofficial].
3.  **Streaming:** Transfer the selected "hot" Gaussians from CPU RAM to the GPU Cache Set. If the cache is full, evict the least recently used (LRU) or least frequently updated Gaussians back to CPU RAM [S1, unofficial].
4.  **Rasterization:** Render the image using the Gaussians currently in the GPU Cache Set via the differentiable rasterizer.
5.  **Gradient Computation:** Compute gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for the rendered image with respect to the active Gaussians.
6.  **Update:** Apply the optimizer step (e.g., Adam) to update the parameters of the active Gaussians.
7.  **Cache Management:** Update the liveness metrics for the updated Gaussians. Gaussians that have not been updated or accessed for a threshold period are marked for eviction [S1, unofficial].
8.  **Repeat:** Return to Step 2 for the next training iteration.

**Pitfalls:**
*   **Thrashing:** If the working set of active Gaussians exceeds the GPU cache size, frequent evictions and re-loading can negate performance gains [G].
*   **Boundary Conditions:** Gaussians on the edge of the frustum may flicker in and out of the cache, causing instability in gradients [G].

## Subtopics

### Out-of-Core Training and Rendering
A framework that stores the full scene in CPU RAM and dynamically streams only visible Gaussians to GPU memory, enabling training and rendering of scenes exceeding VRAM on a single consumer GPU [S2, unofficial]. It uses a lightweight caching system to track recently used Gaussians and reduce CPU-GPU data transfer [S2, unofficial]. A key pitfall is that a single distant view may require access to the full scene, causing high transfer overhead [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the cache hit rate. Best case is $O(1)$ for cache hits; worst case is $O(N)$ for full scene reloads [G].
*   **Memory Complexity:** $O(K)$ on GPU, where $K$ is the cache size, and $O(N)$ on CPU, where $N$ is the total number of Gaussians [S1, unofficial].
*   **Performance:** Reduces GPU memory barrier overhead by minimizing unnecessary data transfers [S1, unofficial].
*   **Strengths:** Enables training on hardware with limited VRAM without complex spatial partitioning [S1, unofficial].
*   **Limitations:** Performance degrades if the temporal coherence of Gaussian updates is low [G].

## Background
CLM was introduced as part of the broader effort to optimize 3D Gaussian Splatting training efficiency [S1, unofficial]. It addresses the limitations of standard rasterization pipelines where all Gaussians must reside in GPU memory, leading to memory bottlenecks and synchronization barriers [S1, unofficial]. The method builds on the principles of dynamic memory management and caching policies commonly used in computer graphics and database systems [G].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
