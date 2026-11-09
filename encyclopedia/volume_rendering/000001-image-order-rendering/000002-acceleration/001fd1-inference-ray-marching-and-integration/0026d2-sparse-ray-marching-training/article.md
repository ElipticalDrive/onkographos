# Sparse Ray Marching (Training)
## Overview
Sparse Ray Marching (Training) is a GPU kernel that samples points along rays during the training phase of neural volume rendering [S1, unofficial]. It efficiently traverses a hashed density grid to identify occupied regions while adapting step sizes and applying perturbation to maintain sampling quality [S1, unofficial]. It differs from inference marchers by prioritizing training stability and memory efficiency through adaptive step sizing and perturbation, and from static allocation methods by dynamically managing sample buffers based on occupancy [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
- `rays_alive`: Array of active ray indices [S1, unofficial].
- `rays_t`: Current distance along each ray [S1, unofficial].
- `ox, oy, oz`: Ray origins [S1, unofficial].
- `dx, dy, dz`: Ray directions [S1, unofficial].
- `far`: Far plane distance [S1, unofficial].
- `bound`: Spatial extent of the grid (half-width) [S1, unofficial].
- `rbound`: Inverse of `bound` ($1/\text{bound}$) [S1, unofficial].
- `H`: Resolution of the cubic density grid ($H \times H \times H$) [S1, unofficial].
- `density_thresh`: Threshold for considering a voxel occupied [S1, unofficial].
- `dt_min`, `dt_max`: Minimum and maximum step sizes [S1, unofficial].
- `dt_gamma`: Step size scaling factor [S1, unofficial].
- `perturb`: Boolean flag for random jittering [S1, unofficial].
- `mean_density`: Average grid density used as a fallback threshold [S1, unofficial].
- `n_alive`: Number of active rays [S1, unofficial].
- `n_step`: Maximum steps per ray [S1, unofficial].
- `grid`: Hashed density grid data structure [S1, unofficial].

**Outputs:**
- `xyz`: Sampled 3D coordinates [S1, unofficial].
- `dir`: Ray directions at samples [S1, unofficial].
- `delta`: Step sizes and cumulative distances [S1, unofficial].

### Data Structures
- **Density Grid**: A 1D array representing a 3D grid of size $H^3$ [S1, unofficial]. Accessed via linear index $idx = nx \cdot H^2 + ny \cdot H + nz$ [S1, unofficial].
- **Ray State**: Managed per-thread, tracking current $t$, step count, and alive status [S1, unofficial].

### Algorithm Steps
1. **Initialization**: Assign thread index $n$. If $n \ge n\_alive$, terminate [S1, unofficial]. Load ray index from `rays_alive[n]` and current distance $t$ from `rays_t[n]` [S1, unofficial].
2. **Perturbation**: If `perturb` is true, initialize a PCG32 random number generator with seed $(n, \text{perturb})$ and add $dt\_min \cdot \text{rng.next\_float()}$ to $t$ [S1, unofficial].
3. **Marching Loop**: While $t < \text{far}$ and step count $< n\_step$:
   - **Coordinate Clamping**: Calculate current point $(x, y, z)$ by clamping $ox + t \cdot dx$, $oy + t \cdot dy$, $oz + t \cdot dz$ to $[-\text{bound}, \text{bound}]$ [S1, unofficial].
   - **Grid Index Mapping**: Map coordinates to grid indices $(nx, ny, nz)$ by clamping $\lfloor 0.5 \cdot (\text{coord} \cdot \text{rbound} + 1) \cdot H \rfloor$ to $[0, H-1]$ [S1, unofficial].
   - **Density Query**: Query grid density at index $nx \cdot H^2 + ny \cdot H + nz$ [S1, unofficial].
   - **Occupancy Check**: If density $> \text{density\_thresh}$: write $x, y, z$ to `xyz`, $dx, dy, dz$ to `dir`, and calculate $dt = \text{clamp}(t \cdot dt\_gamma, dt\_min, dt\_max)$ [S1, unofficial]. Record $dt$ in `delta[0]` and $t - \text{last\_t}$ in `delta[1]` [S1, unofficial]. Update $\text{last\_t} = t$, advance $t \leftarrow t + dt$, increment step counter and output pointers [S1, unofficial].
   - **Empty Space**: Else, calculate distance to next voxel boundary in $x, y, z$ using $\text{signf}(dx), \text{signf}(dy), \text{signf}(dz)$ [S1, unofficial]. Compute $tt = t + \max(0, \min(tx, ty, tz))$ [S1, unofficial]. Advance $t$ in a do-while loop until $t \ge tt$, using the same $dt$ calculation as occupied space [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def signf(x):
    return 1.0 if x >= 0.0 else -1.0

class PCG32:
    def __init__(self, seed):
        self.state = seed & 0xFFFFFFFF
    def next_float(self):
        self.state = (self.state * 6364136223846793005 + 1442695040883263533) & 0xFFFFFFFFFFFFFFFF
        return ((self.state >> 33) & 0xFFFFFFFF) / 4294967296.0

def sparse_ray_marching_train(
    rays_alive, rays_t, ox, oy, oz, dx, dy, dz,
    far, bound, rbound, H, density_thresh,
    dt_min, dt_max, dt_gamma, perturb, mean_density,
    n_alive, n_step, grid, out_xyz, out_dir, out_delta
):
    # n is provided by the execution context (e.g., CUDA thread ID)
    n = 0  # Placeholder for runtime thread index
    if n >= n_alive:
        return

    ray_idx = rays_alive[n]
    t = rays_t[n]
    last_t = t

    if perturb:
        rng = PCG32(seed=(n, perturb))
        t += dt_min * rng.next_float()

    step = 0
    while t < far and step < n_step:
        x = max(-bound, min(bound, ox + t * dx))
        y = max(-bound, min(bound, oy + t * dy))
        z = max(-bound, min(bound, oz + t * dz))

        nx = max(0, min(H - 1, math.floor(0.5 * (x * rbound + 1) * H)))
        ny = max(0, min(H - 1, math.floor(0.5 * (y * rbound + 1) * H)))
        nz = max(0, min(H - 1, math.floor(0.5 * (z * rbound + 1) * H)))

        idx = nx * H * H + ny * H + nz
        density = grid[idx]

        if density > density_thresh:
            out_xyz.append((x, y, z))
            out_dir.append((dx, dy, dz))
            
            dt = max(dt_min, min(dt_max, t * dt_gamma))
            out_delta.append((dt, t - last_t))
            
            last_t = t
            t += dt
            step += 1
        else:
            tx = signf(dx) / dx if dx != 0.0 else float('inf')
            ty = signf(dy) / dy if dy != 0.0 else float('inf')
            tz = signf(dz) / dz if dz != 0.0 else float('inf')
            
            tt = t + max(0.0, min(tx, ty, tz))
            
            while True:
                dt = max(dt_min, min(dt_max, t * dt_gamma))
                t += dt
                if t >= tt:
                    break
                if t == last_t:
                    break
            last_t = t

    rays_t[n] = t
```

### Parameters
- `N_THREAD`: 256 (CUDA block size) [S1, unofficial].
- `perturb`: `uint32_t` boolean flag [S1, unofficial].
- `bound`: Spatial extent of the grid [S1, unofficial].
- `H`: Resolution of the cubic density grid [S1, unofficial].
- `mean_density`: Fallback density threshold [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Floating-point errors in boundary calculations can cause infinite loops or missed samples [G]. The progress guard in the empty-space loop prevents stagnation [G].
- **Boundaries**: Clamping prevents out-of-bounds memory access in the grid [S1, unofficial].
- **Degenerate Input**: Zero-length rays or directions with zero components require careful handling of boundary distances (e.g., using infinity) [G].

## Subtopics

### [CUDA Ray Marching (IDDR-NGP Backend)](5788-cuda-ray-marching-iddr-ngp-backend.md)
A GPU-accelerated ray marching kernel that samples points along rays using a hashed density grid, supporting adaptive step sizes and perturbation. It builds on [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md) [G].

### [Training Ray Marching (Forward Only)](6205-training-ray-marching-forward-only.md)
Training Ray Marching is a training-specific ray marching function that estimates the maximum number of points to allocate based on the mean count from previous epochs, enabling faster training by avoiding over-allocation, while supporting a force-all-rays mode for stability. It differs from inference-only marchers by maintaining state (step counters) across iterations to adaptively tune memory usage, and from static allocation methods by using historical data to predict resource needs.

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of active rays and the number of steps per ray [G]. Sparse traversal reduces the number of density queries in empty space [G].
- **Memory Complexity**: Depends on the number of samples generated. The grid memory is $O(H^3)$ [G].
- **Performance**: Optimized for GPU execution with 256 threads per block [S1, unofficial]. Perturbation reduces banding artifacts but adds RNG overhead [S1, unofficial].
- **Strengths**: Efficient handling of sparse scenes, adaptive step sizes, reduced memory usage compared to uniform sampling [G].
- **Limitations**: Performance degrades if the grid is dense or if the step size parameters are poorly tuned [G].
- **When to Use**: Training phase of neural volume rendering where memory efficiency and sampling quality are critical [G].
- **When Not to Use**: Inference scenarios requiring deterministic, fixed-step sampling or when the grid is extremely dense [G].

## Background
Originating from the Instant Neural Graphics Primitives (NGP) framework, this method was developed to enable fast training of neural radiance fields [S1, unofficial]. It leverages hashed encoding and sparse ray marching to achieve real-time performance on consumer hardware [S1, unofficial]. The method is widely used in modern neural rendering pipelines for training [S1, unofficial].

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
