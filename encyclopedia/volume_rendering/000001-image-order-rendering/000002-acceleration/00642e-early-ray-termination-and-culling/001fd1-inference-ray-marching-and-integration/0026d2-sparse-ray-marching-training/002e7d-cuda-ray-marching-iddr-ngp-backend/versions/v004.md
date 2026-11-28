# CUDA Ray Marching (IDDR-NGP Backend)

## Overview
A GPU-accelerated ray marching kernel that samples points along rays using a hashed density grid, supporting adaptive step sizes and perturbation [S1, unofficial]. It serves as the core inference and training step for the IDDR-NGP backend, differing from standard uniform ray marching by integrating spherical harmonic (SH) evaluation and adaptive step sizing within a single warp-synchronized launch, and from dense voxel tracing by operating on a sparse grid structure with specific boundary handling [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `rays_o`: Origin positions, shape `[N, 3]`, type `float32`.
    *   `rays_d`: Direction vectors, shape `[N, 3]`, type `float32`.
    *   `near`: Near plane distances, shape `[N]`, type `float32`.
    *   `far`: Far plane distances, shape `[N]`, type `float32`.
    *   `density_bitfield`: Hashed density grid lookup table, type `uint8` [S1, unofficial].
*   **Outputs**:
    *   `xyzs`: Sampled 3D coordinates, shape `[N, n_step, 3]`, type `float32`.
    *   `dirs`: Sampled direction vectors, shape `[N, n_step, 3]`, type `float32`.
    *   `deltas`: Step sizes, shape `[N, n_step, 2]`. The first channel stores deltas for RGB integration, the second for depth integration [S1, unofficial].

**Data Structures**
The `density_bitfield` is a `uint8` tensor storing the hashed density grid. It is used for fast lookup during ray marching [S1, unofficial]. The exact hashing scheme (e.g., hash function, grid resolution, bound) is not specified in the provided sources [S1, unofficial].

**Algorithm Steps**
1.  **Initialization**: For each ray $i \in [0, N)$, initialize current position $p = \text{rays\_o}[i]$ and step counter $k=0$.
2.  **Marching Loop**: While $k < n_{\text{max\_steps}}$ and $p$ is within bounds:
    a. **Query Density**: Look up density at $p$ using `density_bitfield`. [G] The specific hash lookup function is not defined in sources.
    b. **Adaptive Step Size**: Calculate step size $\Delta t$ based on the queried density and a threshold `density_thresh`. [G] The exact formula for adaptive stepping is not provided in sources.
    c. **Perturbation**: Apply perturbation to the step or position. [G] Details on perturbation method are not provided in sources.
    d. **Update Position**: $p \leftarrow p + \Delta t \cdot \text{rays\_d}[i]$.
    e. **Store Sample**: Store $p$, $\text{rays\_d}[i]$, and $\Delta t$ in `xyzs[i, k]`, `dirs[i, k]`, and `deltas[i, k]`.
    f. **Increment**: $k \leftarrow k + 1$.
3.  **Completion**: Return `xyzs`, `dirs`, and `deltas`.

**Parameters**
*   `density_thresh`: Density threshold for stopping or adaptive stepping. Typical value: Not given in the sources.
*   `bound`: Scene bound. Typical value: Not given in the sources.
*   `min_near`: Minimum near plane distance. Typical value: Not given in the sources.
*   `bg_radius`: Background radius. Typical value: Not given in the sources.

**Edge Cases and Pitfalls**
*   **Precision**: Floating-point precision in position updates may cause artifacts. [G] No specific mitigation is mentioned.
*   **Boundaries**: Rays may exit the `bound` or `bg_radius`. Specific boundary handling is mentioned but not detailed [S1, unofficial].
*   **Degenerate Input**: Rays with zero length or invalid directions should be handled. [G] No specific handling is mentioned.

## Subtopics
## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: GPU-accelerated, sparse grid support, adaptive step sizing, integrated SH evaluation [S1, unofficial].
*   **Limitations**: Sparse grid structure complexity, dependency on `density_bitfield` layout [S1, unofficial].
*   **When to Use**: Inference and training for IDDR-NGP backend [S1, unofficial].
*   **When Not to Use**: Dense voxel scenes, non-CUDA hardware [S1, unofficial].

## Background
Origin: IDDR-NGP project [S1, unofficial].
History: Part of the NeRF-Factory/IDDR-NGP repository [S1, unofficial].
Usage: Used in the NeRF-Factory/IDDR-NGP project for ray marching [S1, unofficial].

## Sources
[S1] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
