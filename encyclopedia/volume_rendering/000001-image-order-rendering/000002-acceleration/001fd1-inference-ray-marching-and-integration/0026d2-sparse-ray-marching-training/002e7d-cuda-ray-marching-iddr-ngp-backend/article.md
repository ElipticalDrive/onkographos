# CUDA Ray Marching (IDDR-NGP Backend)

## Overview
A GPU-accelerated ray marching kernel that samples points along rays using a hashed density grid, supporting adaptive step sizes and perturbation [S1, unofficial]. It differs from standard uniform ray marching by integrating adaptive stepping and perturbation directly into the sampling loop, and from dense voxel tracing by operating on a sparse hashed grid structure [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `rays_o`: Origin positions, shape `[N, 3]`, type `float32` [S1, unofficial].
    *   `rays_d`: Direction vectors, shape `[N, 3]`, type `float32` [S1, unofficial].
    *   `near`: Near plane distances, shape `[N]`, type `float32` [S1, unofficial].
    *   `far`: Far plane distances, shape `[N]`, type `float32` [S1, unofficial].
    *   `density_bitfield`: Hashed density grid lookup table, type `uint8` [S1, unofficial].
*   **Outputs**:
    *   `xyzs`: Sampled 3D coordinates, shape `[N, n_step, 3]`, type `float32` [S1, unofficial].
    *   `dirs`: Sampled direction vectors, shape `[N, n_step, 3]`, type `float32` [S1, unofficial].
    *   `deltas`: Step sizes, shape `[N, n_step, 2]` where the first channel stores deltas for RGB integration and the second for depth integration [S1, unofficial].

**Data Structures**
The `density_bitfield` is a `uint8` tensor storing the hashed density grid used for fast lookup during ray marching [S1, unofficial]. The exact hashing scheme (e.g., hash function, grid resolution, bound) is not specified in the provided sources [S1, unofficial].

**Algorithm Steps**
1.  **Initialization**: For each ray $i \in [0, N)$, initialize current position $p = \text{rays\_o}[i]$, accumulated distance $t = 0$, and step counter $k = 0$ [G].
2.  **Marching Loop**: While $k < n_{\text{max\_steps}}$ and $t < \text{far}[i] - \text{near}[i]$:
    a. **Query Density**: Look up density at $p$ using `density_bitfield` [Not given in the sources].
    b. **Adaptive Step Size**: Calculate step size $\Delta t$ based on the queried density and `density_thresh` [Not given in the sources].
    c. **Perturbation**: Apply perturbation to the step or position [Not given in the sources].
    d. **Update Position**: $p \leftarrow p + \Delta t \cdot \text{rays\_d}[i]$ and $t \leftarrow t + \Delta t$ [G].
    e. **Store Sample**: Store $p$, $\text{rays\_d}[i]$, and $\Delta t$ in `xyzs[i, k]`, `dirs[i, k]`, and `deltas[i, k]` (both channels set to $\Delta t$) [G].
    f. **Increment**: $k \leftarrow k + 1$ [G].
3.  **Completion**: Return `xyzs`, `dirs`, and `deltas` [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def march_rays(rays_o, rays_d, near, far, density_bitfield, density_thresh, bound, min_near, bg_radius, max_steps):
    N = rays_o.shape[0]
    xyzs = torch.empty(N, max_steps, 3)
    dirs = torch.empty(N, max_steps, 3)
    deltas = torch.empty(N, max_steps, 2)
    for i in range(N):
        if far[i] <= near[i]:
            continue
        p = rays_o[i].clone()
        k = 0
        t = 0.0
        while k < max_steps and t < (far[i] - near[i]):
            # Query density at p using density_bitfield [Not given in the sources.]
            density = 0.0
            # Compute adaptive step size dt based on density and density_thresh [Not given in the sources.]
            dt = 0.0
            # Apply perturbation to p or dt [Not given in the sources.]
            p = p + dt * rays_d[i]
            xyzs[i, k] = p
            dirs[i, k] = rays_d[i]
            deltas[i, k, 0] = dt
            deltas[i, k, 1] = dt
            t += dt
            k += 1
    return xyzs, dirs, deltas
```

**Parameters**
*   `density_thresh`: Density threshold for stopping or adaptive stepping. Typical value: Not given in the sources.
*   `bound`: Scene bound. Typical value: Not given in the sources.
*   `min_near`: Minimum near plane distance. Typical value: Not given in the sources.
*   `bg_radius`: Background radius. Typical value: Not given in the sources.

**Edge Cases and Pitfalls**
*   **Precision**: Floating-point precision in position updates may cause artifacts [G]. No specific mitigation is mentioned [G].
*   **Boundaries**: Rays may exit the `bound` or `bg_radius` [S1, unofficial]. Specific boundary handling is not detailed in the sources [S1, unofficial].
*   **Degenerate Input**: Rays with zero length or invalid directions should be handled [G]. No specific handling is mentioned [G].

## Subtopics

### [Ray Tracing Kernel](5968-ray-tracing-kernel.md)
A CUDA compute kernel that performs forward and backward volumetric ray marching through a sparse hashed density grid using spherical harmonics for color representation. It differs from standard uniform ray marching by utilizing warp-level parallelism for SH evaluation and adaptive step sizes, and from dense voxel cone tracing by operating on a sparse grid structure with specific boundary handling. Choose it when implementing the core inference and training step for the IDDR-NGP backend.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Not given in the sources.
*   **Limitations**: Not given in the sources.
*   **When to Use**: Not given in the sources.
*   **When Not to Use**: Not given in the sources.

## Background
*   **Origin**: Not given in the sources.
*   **History**: Not given in the sources.
*   **Usage**: Not given in the sources.

## Sources
[S1] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
