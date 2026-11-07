# Adaptive Ray Marching with Voxel Skipping

Also called: Sparse Ray Marching

## Overview
Adaptive Ray Marching with Voxel Skipping is a visibility culling technique for direct volume rendering that accelerates ray marching by dynamically adjusting the step size based on local density. It solves the problem of excessive sampling in large empty volumes by using a low-precision traversal algorithm for empty regions and switching to a high-precision algorithm when approaching data. It differs from standard empty-space skipping by dynamically adjusting precision based on proximity to data, rather than using a static skip distance or hierarchical structure like [Adaptive Ray Marching with Macrocell DDA](3869-adaptive-ray-marching-with-macrocell-dda.md).

## How it works
The method operates in two passes to decouple step counting from sample storage, allowing for compact output buffers. It uses a single-resolution density grid to determine occupancy and adapts the step size exponentially in occupied regions while skipping linearly in empty regions.

### Inputs and Outputs
**Inputs:**
- `rays_o`: Ray origins, shape `[N, 3]`.
- `rays_d`: Ray directions, shape `[N, 3]`.
- `grid`: 3D density grid of shape `[H, H, H]`, flattened to 1D. Accessed via linear index `nx * H * H + ny * H + nz`.
- `bound`: Half-width of the cubic scene volume `[-bound, bound]`.
- `H`: Grid resolution (assumed cubic).

**Outputs:**
- `xyzs`: Sampled point positions, shape `[M, 3]`.
- `dirs`: Ray directions at sampled points, shape `[M, 3]`.
- `deltas`: Step sizes (distances) at sampled points, shape `[M]`.
- `rays`: Ray metadata, shape `[N, 3]`. Each row `i` contains `[ray_index, start_point_index, num_steps]`.

### Data Structures
- **Grid**: A 3D array of floats representing density. The source uses a fixed resolution `H=128` for simplicity, though the algorithm is resolution-agnostic.
- **Rays Metadata**: A 1D array of size `3*N` storing ray index, starting index in `xyzs`, and number of steps.

### Parameters
- `DENSITY_THRESH`: Density threshold for occupancy. Default `10.0f`. [S1, unofficial]
- `MAX_STEPS`: Maximum steps per ray. Default `1024`. [S1, unofficial]
- `MIN_STEPSIZE`: Minimum step size in world units. Calculated as `2 * SQRT3() / MAX_STEPS()`. Represents the diagonal of the smallest voxel in a unit cube divided by max steps. Actual `dt_min` is `MIN_STEPSIZE() * bound`. [S1, unofficial]
- `MIN_NEAR`: Minimum near clipping distance. Default `0.05f`. [S1, unofficial]
- `DT_GAMMA`: Gamma factor for step size scaling. Default `1.f / 256.f`. [S1, unofficial]
- `perturb`: Boolean flag. If true, adds random jitter to starting position `t0` using a PCG32 random number generator seeded by ray index `n`. [S1, unofficial]

### Algorithm Steps

1.  **Ray-Box Intersection**:
    For each ray, calculate near and far intersection distances with the bounding box `[-bound, bound]`.
    -   Compute inverse direction: `rdx = 1.0 / rays_d[n][0]`, etc.
    -   For each axis `i`:
        -   `near_i = (-bound - rays_o[n][i]) * rdx_i`
        -   `far_i = (bound - rays_o[n][i]) * rdx_i`
        -   If `near_i > far_i`, swap them.
    -   `t_near = max(near_x, near_y, near_z, MIN_NEAR)`
    -   `t_far = min(far_x, far_y, far_z)`
    -   If `t_near >= t_far`, the ray misses the volume; skip.

2.  **Initialize Step Limits**:
    -   `dt_min = MIN_STEPSIZE() * bound`
    -   `dt_max = 2 * bound / (H - 1)`
    -   `dt_gamma = DT_GAMMA()` if `bound > 1`, else `0.0f`.

3.  **First Pass: Estimate Steps**:
    Initialize `t = t_near`. Loop while `t < t_far` and `num_steps < MAX_STEPS`.
    -   Calculate current position: `x = rays_o[n][0] + t * rays_d[n][0]`, etc.
    -   Clamp position to `[-bound, bound]`.
    -   Map to grid indices:
        -   `rbound = 1.0 / bound`
        -   `nx = clamp(0.5 * (x * rbound + 1) * H, 0.0, H - 1)`
        -   `ny = clamp(0.5 * (y * rbound + 1) * H, 0.0, H - 1)`
        -   `nz = clamp(0.5 * (z * rbound + 1) * H, 0.0, H - 1)`
    -   Query density: `density = grid[nx * H * H + ny * H + nz]`.
    -   **If `density > DENSITY_THRESH`**:
        -   Increment `num_steps`.
        -   Advance `t` by `dt = clamp(t * dt_gamma, dt_min, dt_max)`.
    -   **Else (Empty Space)**:
        -   Calculate distance to next voxel boundary `tt`.
        -   Advance `t` in small steps (unit steps or minimal increments) until `t >= tt`. [S1, unofficial]

4.  **Allocate Output Space**:
    -   `point_index = atomicAdd(counter, num_steps)`
    -   `ray_index = atomicAdd(counter + 1, 1)`
    -   Store metadata:
        -   `rays[ray_index * 3] = n`
        -   `rays[ray_index * 3 + 1] = point_index`
        -   `rays[ray_index * 3 + 2] = num_steps`

5.  **Second Pass: Write Samples**:
    Reset `t = t_near`. Initialize `step = 0`. Loop while `t < t_far` and `step < num_steps`.
    -   Calculate position, grid indices, and density (same as Step 3).
    -   **If `density > DENSITY_THRESH`**:
        -   Write `x, y, z` to `xyzs[point_index + step]`.
        -   Write `rays_d[n]` to `dirs[point_index + step]`.
        -   Write `dt` to `deltas[point_index + step]`.
        -   Advance `t` by `dt = clamp(t * dt_gamma, dt_min, dt_max)`.
        -   Increment `step`.
    -   **Else (Empty Space)**:
        -   Skip to next voxel boundary `tt` using the same logic as Step 3.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def adaptive_ray_marching(rays_o, rays_d, grid, H, bound, perturb=False):
    """
    Adaptive Ray Marching with Voxel Skipping.
    
    Args:
        rays_o: [N, 3] array of ray origins.
        rays_d: [N, 3] array of ray directions.
        grid: [H, H, H] array of densities (flattened access).
        H: Grid resolution.
        bound: Half-width of the scene box [-bound, bound].
        perturb: Whether to perturb ray origins.
        
    Returns:
        xyzs: [M, 3] sampled positions.
        dirs: [M, 3] sampled directions.
        deltas: [M] step sizes.
        rays_meta: [N, 3] ray metadata [ray_idx, start_idx, num_steps].
    """
    N = rays_o.shape[0]
    
    # Constants
    DENSITY_THRESH = 10.0
    MAX_STEPS = 1024
    MIN_NEAR = 0.05
    DT_GAMMA = 1.0 / 256.0
    SQRT3 = math.sqrt(3.0)
    MIN_STEPSIZE_VAL = 2 * SQRT3 / MAX_STEPS
    
    # Precompute bounds
    rbound = 1.0 / bound
    dt_min = MIN_STEPSIZE_VAL * bound
    dt_max = 2 * bound / (H - 1)
    dt_gamma = DT_GAMMA if bound > 1 else 0.0
    
    # Storage for metadata and counts
    rays_meta = np.zeros((N, 3), dtype=np.int32)
    step_counts = np.zeros(N, dtype=np.int32)
    
    # First Pass: Count steps
    for n in range(N):
        ox, oy, oz = rays_o[n]
        dx, dy, dz = rays_d[n]
        
        # Inverse direction
        rdx = 1.0 / dx if dx != 0 else float('inf')
        rdy = 1.0 / dy if dy != 0 else float('inf')
        rdz = 1.0 / dz if dz != 0 else float('inf')
        
        # Intersection with [-bound, bound]
        near_x = (-bound - ox) * rdx
        far_x = (bound - ox) * rdx
        if near_x > far_x: near_x, far_x = far_x, near_x
        
        near_y = (-bound - oy) * rdy
        far_y = (bound - oy) * rdy
        if near_y > far_y: near_y, far_y = far_y, near_y
        
        near_z = (-bound - oz) * rdz
        far_z = (bound - oz) * rdz
        if near_z > far_z: near_z, far_z = far_z, near_z
        
        t_near = max(near_x, near_y, near_z, MIN_NEAR)
        t_far = min(far_x, far_y, far_z)
        
        if t_near >= t_far:
            step_counts[n] = 0
            continue
            
        t = t_near
        num_steps = 0
        
        # Perturbation [G]
        if perturb:
            # Simple jitter implementation [G]
            # In practice, use PCG32 seeded by n
            t += (np.random.rand() - 0.5) * dt_min 
            
        while t < t_far and num_steps < MAX_STEPS:
            # Current position
            x = ox + t * dx
            y = oy + t * dy
            z = oz + t * dz
            
            # Clamp to bounds
            x = max(-bound, min(bound, x))
            y = max(-bound, min(bound, y))
            z = max(-bound, min(bound, z))
            
            # Map to grid indices
            nx = int(max(0, min(H - 1, 0.5 * (x * rbound + 1) * H)))
            ny = int(max(0, min(H - 1, 0.5 * (y * rbound + 1) * H)))
            nz = int(max(0, min(H - 1, 0.5 * (z * rbound + 1) * H))
            
            # Query density
            density = grid[nx * H * H + ny * H + nz]
            
            if density > DENSITY_THRESH:
                num_steps += 1
                # Adaptive step size
                dt = t * dt_gamma
                dt = max(dt_min, min(dt_max, dt))
                t += dt
            else:
                # Skip to next voxel boundary [S1, unofficial]
                # Calculate distance to next boundary for each axis
                # This is a simplified linear skip [G]
                # Exact implementation requires calculating t_next for each axis crossing
                # and taking the minimum positive t_next > t
                t_next_x = float('inf')
                t_next_y = float('inf')
                t_next_z = float('inf')
                
                if dx > 0:
                    t_next_x = ((nx + 1) / (0.5 * rbound * H) - 1) * 0.5 * bound - ox) / dx
                elif dx < 0:
                    t_next_x = ((nx) / (0.5 * rbound * H) - 1) * 0.5 * bound - ox) / dx
                
                if dy > 0:
                    t_next_y = ((ny + 1) / (0.5 * rbound * H) - 1) * 0.5 * bound - oy) / dy
                elif dy < 0:
                    t_next_y = ((ny) / (0.5 * rbound * H) - 1) * 0.5 * bound - oy) / dy
                    
                if dz > 0:
                    t_next_z = ((nz + 1) / (0.5 * rbound * H) - 1) * 0.5 * bound - oz) / dz
                elif dz < 0:
                    t_next_z = ((nz) / (0.5 * rbound * H) - 1) * 0.5 * bound - oz) / dz
                
                t_next = min(t_next_x, t_next_y, t_next_z)
                if t_next > t:
                    t = t_next
                else:
                    t += dt_min # Prevent infinite loop [G]
                    
        step_counts[n] = num_steps

    # Allocate output buffers
    M = sum(step_counts)
    xyzs = np.zeros((M, 3))
    dirs = np.zeros((M, 3))
    deltas = np.zeros(M)
    
    # Compute start indices using cumulative sum
    start_indices = np.zeros(N, dtype=np.int32)
    if N > 0:
        start_indices[1:] = np.cumsum(step_counts[:-1])
        
    # Second Pass: Write samples
    for n in range(N):
        if step_counts[n] == 0:
            continue
            
        ox, oy, oz = rays_o[n]
        dx, dy, dz = rays_d[n]
        
        rdx = 1.0 / dx if dx != 0 else float('inf')
        rdy = 1.0 / dy if dy != 0 else float('inf')
        rdz = 1.0 / dz if dz != 0 else float('inf')
        
        near_x = (-bound - ox) * rdx
        far_x = (bound - ox) * rdx
        if near_x > far_x: near_x, far_x = far_x, near_x
        
        near_y = (-bound - oy) * rdy
        far_y = (bound - oy) * rdy
        if near_y > far_y: near_y, far_y = far_y, near_y
        
        near_z = (-bound - oz) * rdz
        far_z = (bound - oz) * rdz
        if near_z > far_z: near_z, far_z = far_z, near_z
        
        t_near = max(near_x, near_y, near_z, MIN_NEAR)
        t_far = min(far_x, far_y, far_z)
        
        t = t_near
        step = 0
        current_idx = start_indices[n]
        
        while t < t_far and step < step_counts[n]:
            x = ox + t * dx
            y = oy + t * dy
            z = oz + t * dz
            
            x = max(-bound, min(bound, x))
            y = max(-bound, min(bound, y))
            z = max(-bound, min(bound, z))
            
            nx = int(max(0, min(H - 1, 0.5 * (x * rbound + 1) * H)))
            ny = int(max(0, min(H - 1, 0.5 * (y * rbound + 1) * H)))
            nz = int(max(0, min(H - 1, 0.5 * (z * rbound + 1) * H)))
            
            density = grid[nx * H * H + ny * H + nz]
            
            if density > DENSITY_THRESH:
                xyzs[current_idx] = [x, y, z]
                dirs[current_idx] = [dx, dy, dz]
                
                dt = t * dt_gamma
                dt = max(dt_min, min(dt_max, dt))
                deltas[current_idx] = dt
                
                t += dt
                current_idx += 1
                step += 1
            else:
                # Skip to next voxel boundary
                t_next_x = float('inf')
                t_next_y = float('inf')
                t_next_z = float('inf')
                
                if dx > 0:
                    t_next_x = (((nx + 1) / (0.5 * rbound * H) - 1) * 0.5 * bound - ox) / dx
                elif dx < 0:
                    t_next_x = (((nx) / (0.5 * rbound * H) - 1) * 0.5 * bound - ox) / dx
                
                if dy > 0:
                    t_next_y = (((ny + 1) / (0.5 * rbound * H) - 1) * 0.5 * bound - oy) / dy
                elif dy < 0:
                    t_next_y = (((ny) / (0.5 * rbound * H) - 1) * 0.5 * bound - oy) / dy
                    
                if dz > 0:
                    t_next_z = (((nz + 1) / (0.5 * rbound * H) - 1) * 0.5 * bound - oz) / dz
                elif dz < 0:
                    t_next_z = (((nz) / (0.5 * rbound * H) - 1) * 0.5 * bound - oz) / dz
                
                t_next = min(t_next_x, t_next_y, t_next_z)
                if t_next > t:
                    t = t_next
                else:
                    t += dt_min
                    
        rays_meta[n] = [n, start_indices[n], step_counts[n]]
        
    return xyzs, dirs, deltas, rays_meta
```

### Edge Cases and Pitfalls
-   **Empty Rays**: Rays that miss the bounding box or have `t_near >= t_far` result in zero steps. The composite kernel should check `if (num_steps == 0)` and return early with zeroed outputs. [S1, unofficial]
-   **Max Steps Exceeded**: If a ray exceeds `MAX_STEPS`, the first pass stops counting. The second pass will only write the counted steps. The composite kernel should check `if (offset + num_steps >= M)` to prevent buffer overflows. [S1, unofficial]
-   **Precision**: The mapping from world coordinates to grid indices uses clamping. Ensure `rbound` and grid indices are computed consistently between passes. [S1, unofficial]
-   **Threshold Selection**: The source notes uncertainty about how to decide `DENSITY_THRESH`. [S1, unofficial]

## Subtopics

### Shadow ray bit-flag acceleration
A technique to eliminate unnecessary shadow ray casts by storing per-voxel, per-light status flags in the spatial grid. Before casting a shadow ray to a light source, check the corresponding bit flag in the current voxel; if the flag indicates full shadow or clear visibility, skip the ray cast. Each voxel stores two bits per light source: bit 0 indicates the voxel is completely in shadow, bit 1 indicates the voxel is unobstructed from the light source. [S2, unofficial]

## Operating characteristics
-   **Time Complexity**: The first pass is $O(N \cdot S_{empty} + N \cdot S_{occupied})$ where $S_{empty}$ is the number of empty steps skipped and $S_{occupied}$ is the number of occupied steps. The second pass is proportional to the total number of samples $M$.
-   **Memory Complexity**: $O(M)$ for output buffers, where $M$ is the total number of samples. $O(H^3)$ for the density grid.
-   **Performance**: The method reduces the number of samples significantly in empty regions by skipping to voxel boundaries. In occupied regions, it uses adaptive step sizes to reduce samples in smooth areas. [S1, unofficial]
-   **Strengths**: Efficient for scenes with large empty spaces. Simple implementation. [S1, unofficial]
-   **Limitations**: Requires two passes. The density grid must be updated regularly. The threshold `DENSITY_THRESH` is heuristic. [S1, unofficial]

## Background
The method is part of the Instant-NGP implementation, specifically the `march_rays_train` kernel. It is designed for training neural radiance fields (NeRF) with hash encodings. The approach builds on [Resumable Ray Marching Iterator](1505-resumable-ray-marching-iterator.md). [S1, unofficial]

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S2] Fast Voxel Traversal for Ray Tracing | PDF, https://www.scribd.com/document/6913705/A-faster-voxel-traversal-algorithm-for-ray-tracing, fetched 2026-10-25 (unofficial)
