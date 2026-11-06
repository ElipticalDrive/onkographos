# Gradient-Based Surface Normal Estimation

Also called: CalculateSurfaceNormal, Density Computation & Normal Estimation, Density Field Normal Estimation, Gradient Normal Estimation, Gradient-Based Surface Normal Calculation, Gradient-based normal estimation, Local density field

## Overview
This method computes surface normals for a voxelized volume by deriving the gradient of a locally computed density field on the fly, rather than storing explicit normal vectors or relying on mesh normals [S1, unofficial]. It serves as a high-resolution, coherent normal estimation technique suitable for applications like particle collision response and ray marching, where memory efficiency and speed are prioritized over the global smoothness provided by large-support distance fields [S1, unofficial]. It differs from [SDF Normal Estimation via Finite Differences](4619-sdf-normal-estimation-via-finite-differences.md) by using a direct gradient sampling call or a specific density kernel evaluation instead of manual finite difference offsets, reducing function call overhead and potential numerical instability [S2, unofficial].

## How it works
**Inputs:**
- A point $p$ in 3D space (typically a particle position or ray intersection point).
- A voxelized density field representation (often a 3D texture or bit-packed voxel grid).
- A density estimation kernel size $K$, typically $2^3$ (8 voxels) [S1, unofficial].

**Outputs:**
- A normalized surface normal vector $\mathbf{n}$ at point $p$.
- Optionally, a signed distance value $d$ if the density field approximates a distance field [S3, unofficial].

**Data Structures:**
- **Voxel Grid:** A 3D array or texture storing density values. In GPU implementations, this may be bit-packed to save memory, where a 32-bit integer lookup texture is avoided due to memory constraints ($2^{32}$ entries) [S1, unofficial].
- **Density Kernel:** A small $2 \times 2 \times 2$ neighborhood of voxels used to estimate the local gradient [S1, unofficial].

**Steps:**
1. **Evaluate Density/Distance:** Compute the local density or signed distance $d$ at point $p$. This can be done by evaluating the density field using the kernel or a specific distance evaluator [S3, unofficial].
   $$ d = \text{EvaluateDistanceAt}(p) $$
2. **Estimate Gradient:** Compute the gradient $\mathbf{g}$ of the density field at $p$. This is achieved by sampling the gradient directly from the volume or computing it via finite differences over the small kernel [S2, unofficial].
   $$ \mathbf{g} = \text{EstimateGradient}(p) $$
   *Note:* If using finite differences, a fixed epsilon $\epsilon$ (e.g., $0.0001$) is used for offsets. However, direct gradient sampling is preferred to reduce instability [S2, unofficial].
3. **Normalize:** Normalize the gradient vector to obtain the surface normal $\mathbf{n}$.
   $$ \mathbf{n} = \frac{\mathbf{g}}{\|\mathbf{g}\|} $$
4. **Optional Offset:** If a signed distance $d$ is available, the vector from the point to the closest surface approximation can be computed as:
   $$ \mathbf{v} = d \cdot \mathbf{n} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_normal(p, voxel_grid, kernel_size=2):
    """
    Estimate surface normal at point p using gradient of local density field.
    
    Args:
        p: Vector3, point in space.
        voxel_grid: 3D array or texture representing density/distance field.
        kernel_size: int, size of the density estimation kernel (default 2).
        
    Returns:
        Vector3, normalized surface normal.
    """
    # Step 1: Evaluate local density or distance at p
    # This may involve sampling the voxel grid or using a distance evaluator
    d = evaluate_distance_at(p, voxel_grid)
    
    # Step 2: Estimate gradient
    # Option A: Direct gradient sampling (preferred for stability)
    grad = sample_gradient(p, voxel_grid)
    
    # Option B: Finite differences (if direct sampling unavailable)
    # epsilon = 0.0001
    # grad_x = (evaluate_distance_at(p + Vector3(epsilon, 0, 0), voxel_grid) - 
    #           evaluate_distance_at(p - Vector3(epsilon, 0, 0), voxel_grid)) / (2 * epsilon)
    # grad_y = (evaluate_distance_at(p + Vector3(0, epsilon, 0), voxel_grid) - 
    #           evaluate_distance_at(p - Vector3(0, epsilon, 0), voxel_grid)) / (2 * epsilon)
    # grad_z = (evaluate_distance_at(p + Vector3(0, 0, epsilon), voxel_grid) - 
    #           evaluate_distance_at(p - Vector3(0, 0, epsilon), voxel_grid)) / (2 * epsilon)
    # grad = Vector3(grad_x, grad_y, grad_z)
    
    # Step 3: Normalize
    norm = length(grad)
    if norm < 1e-6:
        return Vector3(0, 1, 0) # Default normal if gradient is zero
    normal = grad / norm
    
    return normal
```

**Parameters:**
- **Kernel Size:** $2^3$ (8 voxels) is sufficient for acceptable quality [S1, unofficial].
- **Voxel Grid Resolution:** Commonly $512^3$ for particle demos [S1, unofficial].
- **Epsilon (if using finite differences):** $0.0001$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision Loss:** Using a fixed epsilon in finite differences can cause precision loss or aliasing if the volume sampling frequency is not aligned with the epsilon scale [S2, unofficial].
- **Hardware Constraints:** DX10 hardware cannot use 32-bit integer lookup textures efficiently ($2^{32}$ entries). Bitwise arithmetic is more efficient than breaking integers into 16-bit parts, which would require 64 lookups per 1024-bit voxel column [S1, unofficial].
- **Degenerate Gradients:** If the gradient magnitude is near zero (e.g., inside a uniform region), normalization fails. A default normal should be returned [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, as it involves a fixed number of voxel samples and arithmetic operations.
- **Memory Complexity:** Depends on the voxel grid resolution. For a $512^3$ grid, memory usage is significant but manageable if bit-packed [S1, unofficial].
- **Performance:** In a particle system collision response test with a $512^3$ voxel grid, timings were 0.32 ms for $256^2$ particles, 1.0 ms for $512^2$ particles, and 4.0 ms for $1024^2$ particles [S1, unofficial].
- **Strengths:** High resolution and coherence with the voxelized volume; does not rely on mesh normals; faster to compute than larger-support distance fields; suitable for applications needing only limited distance information [S1, unofficial].
- **Limitations:** Less general than global distance fields; quality depends on voxel resolution; potential numerical instability if finite differences are used improperly [S1, S2, unofficial].
- **When to Use:** When memory is constrained, high-frequency surface details are needed, or when working with dynamic scenes where explicit normal storage is prohibitive [S1, unofficial].
- **When Not to Use:** When global smoothness or large-scale distance information is required [S1, unofficial].

## Background
This method was presented in the context of single-pass GPU solid voxelization for dynamic scenes [S1, unofficial]. It was developed to provide an efficient way to derive normals and distance-like information from voxelized geometry without the overhead of storing explicit normals or using complex depth peeling techniques [S1, unofficial]. The approach leverages the fact that the gradient of a density field approximates the surface normal, allowing for on-the-fly computation during particle system queries or ray tracing [S1, unofficial].

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
[S2] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
[S3] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
