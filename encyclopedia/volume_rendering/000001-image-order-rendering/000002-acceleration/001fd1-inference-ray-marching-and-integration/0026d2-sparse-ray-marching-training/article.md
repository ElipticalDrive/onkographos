# Sparse Ray Marching (Training)
## Overview
Sparse Ray Marching (Training) is a GPU-accelerated kernel designed to sample points along rays for the training phase of neural volume rendering (e.g., Instant Neural Graphics Primitives). It solves the problem of efficiently traversing a sparse hashed density grid to identify occupied regions while adapting step sizes to maintain sampling quality. It differs from inference marchers by prioritizing training stability and memory efficiency through adaptive step sizing and perturbation, and from static allocation methods by dynamically managing sample buffers based on occupancy.
## How it works
### Inputs and Outputs
**Inputs:**
- `rays_alive`: Array of active ray indices.
- `rays_t`: Current distance along each ray.
- `ox, oy, oz`: Ray origins.
- `dx, dy, dz`: Ray directions.
- `far`: Far plane distance.
- `bound`: Spatial extent of the grid (half-width).
- `rbound`: Inverse of `bound` ($1/bound$).
- `H`: Resolution of the cubic density grid ($H \times H \times H$).
- `density_thresh`: Threshold for considering a voxel occupied.
- `dt_min`, `dt_max`: Minimum and maximum step sizes.
- `dt_gamma`: Step size scaling factor.
- `perturb`: Boolean flag for random jittering.
- `mean_density`: Average grid density (fallback threshold).
- `n_alive`: Number of active rays.
- `n_step`: Maximum steps per ray.
- `grid`: Hashed density grid data structure.

**Outputs:**
- `xyz`: Sampled 3D coordinates.
- `dir`: Ray directions at samples.
- `delta`: Step sizes and cumulative distances.

### Data Structures
- **Density Grid**: A 1D array representing a 3D grid of size $H^3$. Accessed via linear index $idx = nx \cdot H^2 + ny \cdot H + nz$.
- **Ray State**: Managed per-thread or per-batch, tracking current $t$, step count, and alive status.

### Algorithm Steps
1. **Initialization**:
   - Assign thread index $n$. If $n \ge n\_alive$, terminate.
   - Load ray index from `rays_alive[n]` and current distance $t$ from `rays_t[n]`.
   - If `perturb` is true:
     - Initialize a PCG32 random number generator with seed $(n, \text{perturb})$.
     - Add $dt\_min \cdot \text{rng.next\_float()}$ to $t$.

2. **Marching Loop**:
   - While $t < \text{far}$ and step count $< n\_step$:
     - **Coordinate Clamping**:
       - Calculate current point $(x, y, z)$ by clamping:
         $$ x = \text{clamp}(ox + t \cdot dx, -\text{bound}, \text{bound}) $$
         $$ y = \text{clamp}(oy + t \cdot dy, -\text{bound}, \text{bound}) $$
         $$ z = \text{clamp}(oz + t \cdot dz, -\text{bound}, \text{bound}) $$
     - **Grid Index Mapping**:
       - Map coordinates to grid indices $(nx, ny, nz)$:
         $$ nx = \text{clamp}\left( \lfloor 0.5 \cdot (x \cdot \text{rbound} + 1) \cdot H \rfloor, 0, H-1 \right) $$
         $$ ny = \text{clamp}\left( \lfloor 0.5 \cdot (y \cdot \text{rbound} + 1) \cdot H \rfloor, 0, H-1 \right) $$
         $$ nz = \text{clamp}\left( \lfloor 0.5 \cdot (z \cdot \text{rbound} + 1) \cdot H \rfloor, 0, H-1 \right) $$
     - **Density Query**:
       - Compute linear index $idx = nx \cdot H^2 + ny \cdot H + nz$.
       - Query density $d = \text{grid}[idx]$.
     - **Occupancy Check**:
       - If $d > \text{density\_thresh}$:
         - Write $x, y, z$ to `xyz` buffer.
         - Write $dx, dy, dz$ to `dir` buffer.
         - Calculate step size $dt = \text{clamp}(t \cdot dt\_gamma, dt\_min, dt\_max)$.
         - Store $dt$ in `delta[0]` and cumulative distance in `delta[1]`.
         - Update $t \leftarrow t + dt$.
         - Increment step counter and output pointers.
       - Else (empty space):
         - Calculate distance to next voxel boundary in $x, y, z$ using $\text{signf}(dx), \text{signf}(dy), \text{signf}(dz)$.
         - Compute $tt = t + \max(0, \min(tx, ty, tz))$.
         - Advance $t$ in a do-while loop until $t \ge tt$, using the same $dt$ calculation as occupied space.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_ray_marching_train(
    rays_alive, rays_t, ox, oy, oz, dx, dy, dz,
    far, bound, rbound, H, density_thresh,
    dt_min, dt_max, dt_gamma, perturb, mean_density,
    n_alive, n_step, grid
):
    # Assume this runs in a parallel context where 'n' is the thread ID
    n = get_thread_index()
    if n >= n_alive:
        return

    ray_idx = rays_alive[n]
    t = rays_t[n]
    last_t = t

    # Perturbation
    if perturb:
        rng = PCG32(seed=(n, perturb))
        t += dt_min * rng.next_float()

    step = 0
    while t < far and step < n_step:
        # Clamp coordinates to grid bounds
        x = clamp(ox + t * dx, -bound, bound)
        y = clamp(oy + t * dy, -bound, bound)
        z = clamp(oz + t * dz, -bound, bound)

        # Map to grid indices
        nx = clamp(floor(0.5 * (x * rbound + 1) * H), 0, H - 1)
        ny = clamp(floor(0.5 * (y * rbound + 1) * H), 0, H - 1)
        nz = clamp(floor(0.5 * (z * rbound + 1) * H), 0, H - 1)

        # Query density
        idx = nx * H * H + ny * H + nz
        density = grid[idx]

        if density > density_thresh:
            # Occupied: sample and advance
            write_xyz(x, y, z)
            write_dir(dx, dy, dz)
            
            dt = clamp(t * dt_gamma, dt_min, dt_max)
            write_delta(dt, t - last_t)
            
            last_t = t
            t += dt
            step += 1
            advance_output_pointers()
        else:
            # Empty: skip to next voxel boundary
            # Calculate distances to next boundaries
            tx = signf(dx) / dx if dx != 0 else float('inf')
            ty = signf(dy) / dy if dy != 0 else float('inf')
            tz = signf(dz) / dz if dz != 0 else float('inf')
            
            # Note: signf returns 1 or -1. The distance to boundary is derived from grid spacing.
            # Simplified logic based on source:
            tt = t + max(0, min(tx, ty, tz))
            
            # Advance t until it reaches or exceeds the boundary
            while t < tt:
                dt = clamp(t * dt_gamma, dt_min, dt_max)
                t += dt
            
            # Ensure we don't infinite loop if dt is very small
            if t == last_t:
                break
            last_t = t

    rays_t[n] = t
```

### Parameters
- `N_THREAD`: 256 (CUDA block size).
- `perturb`: `uint32_t` boolean.
- `bound`: Spatial extent.
- `H`: Grid resolution.
- `mean_density`: Fallback threshold.

### Edge Cases and Pitfalls
- **Precision**: Floating-point errors in boundary calculations can cause infinite loops or missed samples. The do-while loop for empty space ensures progress.
- **Boundaries**: Clamping prevents out-of-bounds memory access in the grid.
- **Degenerate Input**: Zero-length rays or directions with zero components require careful handling of boundary distances (e.g., using infinity).

## Subtopics

### [CUDA Ray Marching (IDDR-NGP Backend)](5788-cuda-ray-marching-iddr-ngp-backend.md)
A GPU-accelerated ray marching kernel that samples points along rays using a hashed density grid, supporting adaptive step sizes and perturbation. It builds on [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md) [G].

### [Training Ray Marching (Forward Only)](6205-training-ray-marching-forward-only.md)
Training Ray Marching is a specialized ray marching function designed to optimize the training phase of neural volume rendering by dynamically allocating memory for ray samples. It estimates the maximum number of points to allocate based on the mean sample count from previous training epochs, thereby avoiding the computational overhead of over-allocating memory while supporting a "force-all-rays" mode to ensure stability when estimates are insufficient. It differs from inference-only marchers by maintaining state (step counters) across iterations to adaptively tune memory usage, and from static allocation methods by using historical data to predict resource needs.

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of active rays and the number of steps per ray. Sparse traversal reduces the number of density queries in empty space.
- **Memory Complexity**: Depends on the number of samples generated. The grid memory is $O(H^3)$.
- **Performance**: Optimized for GPU execution with 256 threads per block. Perturbation reduces banding artifacts but adds RNG overhead.
- **Strengths**: Efficient handling of sparse scenes, adaptive step sizes, reduced memory usage compared to uniform sampling.
- **Limitations**: Performance degrades if the grid is dense or if the step size parameters are poorly tuned.
- **When to Use**: Training phase of neural volume rendering where memory efficiency and sampling quality are critical.
- **When Not to Use**: Inference scenarios requiring deterministic, fixed-step sampling or when the grid is extremely dense.

## Background
Originating from the Instant Neural Graphics Primitives (NGP) framework, this method was developed to enable fast training of neural radiance fields. It leverages hashed encoding and sparse ray marching to achieve real-time performance on consumer hardware. The method is widely used in modern neural rendering pipelines for training.

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
