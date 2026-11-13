# Recursive Ray Marching with Early Stopping

Also called: Dynamic Synchronized Inference

## Overview
Recursive Ray Marching with Early Stopping is a parallelization technique for sphere tracing that maintains a dynamic mask of unfinished rays to prevent wasted computation on rays that have already converged or exited the bounding volume [S1][S2, unofficial]. It differs from its siblings, such as [Beam Optimization](932-beam-optimization.md) and [Block Culling](5431-block-culling.md), by operating on a per-ray basis within a synchronous batch, dynamically filtering the active set at each iteration rather than using static spatial structures or beam pruning [S1][S2, unofficial].

## How it works
**Inputs:**
- Camera rays (origin $O$, direction $D$) for each pixel.
- Signed Distance Function (SDF) query function $SDF(P)$.
- Marching parameters: `max_steps`, `ratio` (step multiplier), `stop_threshold` (convergence epsilon).
- Optional: `K` (number of queries to retain for backpropagation) [S1].

**Outputs:**
- Rendered depth, color, or normal for each pixel.
- Gradient information for the SDF decoder (if differentiable) [S1].

**Data Structures:**
1.  `marching_zdepth`: Tensor of shape $(H, W)$, initialized to 0. Stores the accumulated distance along each ray.
2.  `unfinished_mask`: Boolean tensor of shape $(H, W)$, initialized to `True` for all pixels within the valid camera frustum.
3.  `query_locations`: List or tensor storing the 3D coordinates $P$ of each SDF query performed, used for gradient retention [S1].
4.  `min_sdf_values`: Tensor storing the minimum absolute SDF value encountered for each ray during the current batch step [S1].

**Algorithm Steps:**

1.  **Initialization:**
    -   Set `marching_zdepth` to zeros.
    -   Set `unfinished_mask` to `valid_mask` (rays within the bounding volume, e.g., unit sphere) [S2, unofficial].
    -   Initialize `query_locations` as empty [S1].

2.  **Iteration Loop:**
    For `step` in range(`max_steps`):
    
    a.  **Select Active Rays:**
        Identify indices where `unfinished_mask` is `True`. Let this set be $A$ [S2, unofficial].
        If $A$ is empty, break loop [G].

    b.  **Batch Query:**
        For each ray $i \in A$:
        -   Calculate current position: $P_i = O_i + \text{marching\_zdepth}_i \cdot D_i$.
        -   Query SDF: $d_i = SDF(P_i)$.
        -   Store $P_i$ in `query_locations` for backpropagation [S1].
        -   Store $|d_i|$ in `min_sdf_values` for this step [S1].

    c.  **Update Step Size:**
        For each ray $i \in A$:
        -   Compute step size: $\Delta z_i = \min(d_i, \text{max\_step\_size}) \cdot \text{ratio}$ [S2, unofficial].
        -   *Note:* The `ratio` parameter (typically 1.2 to 1.8) scales the SDF value to ensure convergence [S1].
        -   Update depth: $\text{marching\_zdepth}_i \leftarrow \text{marching\_zdepth}_i + \Delta z_i$ [S2, unofficial].

    d.  **Convergence Check:**
        For each ray $i \in A$:
        -   Check if converged: $|d_i| < \text{stop\_threshold}$ OR ray is out of bounds (e.g., distance from origin exceeds sphere radius) [S1][S2, unofficial].
        -   If converged or out of bounds, set `unfinished_mask[i] = False` [S1].

    e.  **Gradient Optimization (Differentiable Rendering):**
        To avoid unaffordable memory consumption from storing all query gradients [S1]:
        1.  Detach the tensor graph for `min_sdf_values` [S1].
        2.  Find the minimum absolute SDF value for each ray across all steps performed so far (or within the current batch context) [S1].
        3.  Identify the specific query locations corresponding to these minimum values [S1].
        4.  Re-evaluate the SDF for these specific locations with gradients attached to compute the loss gradient [S1].
        5.  Set `K` to the number of queries to keep. $K=1$ is sufficient for depth rendering; $K=3$ for shape completion [S1].

    f.  **Result Storage:**
        -   For rays in $A$, store the final $P_i$ and $d_i$.
        -   For rays where `unfinished_mask` is `False`, pad results with zeros or previous values to maintain tensor shape [S2, unofficial].

**Parameters:**
-   `max_steps`: Integer, typically 100 [S1].
-   `ratio`: Float, typically 1.2 to 1.8 [S1].
-   `stop_threshold`: Float, typically $5 \times 10^{-5}$ [S2, unofficial].
-   `K`: Integer, 1 for depth, 3 for shape completion [S1].

**Edge Cases:**
-   **Slow Convergence:** When close to the surface, $d_i \approx 0$, causing small steps. The `stop_threshold` prevents infinite loops [S1].
-   **Memory Consumption:** Storing gradients for all queries in high-resolution rendering (e.g., 2048x2048) is prohibitive. The detached tensor minimization trick is required [S1].
-   **Thin Structures:** Large `ratio` values may cause rays to overshoot thin geometry, creating holes. Small `ratio` values cause slow convergence [S1].

## Subtopics

### [Adaptive Ray Marching with Dynamic Step Count](3888-adaptive-ray-marching-with-dynamic-step-count.md)
Adaptive Ray Marching with Dynamic Step Count is an iterative volume rendering algorithm that samples rays until they are occluded or a maximum step count is reached, using a density grid for acceleration. It differs from its parent, [Recursive Ray Marching with Early Stopping](3546-recursive-ray-marching-with-early-stopping.md), by dynamically adjusting the number of steps per ray based on the count of currently active rays.

## Operating characteristics
-   **Time Complexity:** $O(N \cdot S_{active})$, where $N$ is the number of pixels and $S_{active}$ is the average number of steps for active rays. Early stopping reduces $S_{active}$ significantly compared to fixed-step marching when rays converge at different rates [S1].
-   **Memory Complexity:** $O(N \cdot K)$ for gradient storage, where $K$ is the number of retained queries per ray. Without the detached tensor optimization, memory would scale with total steps $S_{total}$ [S1].
-   **Performance:** Enables high-resolution rendering (2048x2048) on single GPUs (e.g., GTX-1080Ti) by avoiding wasted computation on converged rays [S1].
-   **Strengths:** Efficient for scenes with varying depth complexity; differentiable; simple implementation within deep learning frameworks [S1].
-   **Limitations:** Synchronous batching requires all active rays to be processed together, which may lead to load imbalance if active ray counts vary drastically. The detached tensor trick adds implementation complexity [S1].

## Background
The method was introduced as "Dynamic Synchronized Inference" in the paper "DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing" [S1]. It addresses the inefficiency of naive parallel sphere tracing where rays converge at different rates, leading to wasted computation. The implementation details and code examples are available in the unofficial repository [S2, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
