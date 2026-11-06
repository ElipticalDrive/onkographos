# Adaptive Ray Marching with Dynamic Step Count

Also called: CUDA Ray Marching with Adaptive Step Count

## Overview
Adaptive Ray Marching with Dynamic Step Count is a volume rendering technique that optimizes computational load by dynamically adjusting the number of sampling steps per ray based on the number of currently active (unterminated) rays. It solves the problem of redundant computation on rays that have already exited the scene or hit a surface. It differs from its parent, [Recursive Ray Marching with Early Stopping](3546-recursive-ray-marching-with-early-stopping.md), by explicitly batching the remaining active rays into fixed-size chunks of steps to maximize hardware throughput, rather than processing rays individually or with static step counts.

## How it works
The method operates in an iterative loop. In each iteration, it determines how many rays are still "alive" (i.e., have not yet hit a surface or exceeded the maximum depth). It then calculates a batch size of steps to process for these alive rays, ensuring efficient utilization of parallel processing units (such as CUDA cores). The algorithm proceeds by marching these rays, querying densities and colors, and compositing the results.

### Inputs and Outputs
**Inputs:**
- `rays_o`: Ray origins, shape $[B, N, 3]$.
- `rays_d`: Ray directions, shape $[B, N, 3]$.
- `near`, `far`: Near and far clipping planes.
- `self.bound`: Bounding box parameters.
- `self.density_grid`: Acceleration structure (e.g., density grid or hash grid).
- `self.mean_density`: Mean density value for normalization or thresholding.
- `bg_color`: Background color, default 1.0.
- `perturb`: Boolean flag for ray perturbation.
- `alpha_premultiplied`: Boolean flag determining output normalization.
- `B`: Batch size.
- `N`: Number of rays per batch.

**Outputs:**
- `image`: Rendered image, shape $[B, N, 4]$. The first three channels are RGB, and the fourth channel is the accumulated weight sum (transparency mask).

### Data Structures
- `rays_alive`: A ping-pong buffer array of two boolean tensors, shape $[2, B, N]$. `rays_alive[i % 2]` indicates which rays are still active in the current iteration $i$.
- `rays_t`: A ping-pong buffer array of two float tensors, shape $[2, B, N]$. `rays_t[i % 2]` stores the current accumulated distance along the ray for active rays.
- `alive_counter`: An integer scalar tracking the total number of active rays across the batch.
- `weights_sum`: Accumulated weight (opacity) per ray, shape $[B, N]$.
- `depth`: Accumulated depth per ray, shape $[B, N]$.

### Parameters
- `n_step`: Number of steps to process per alive ray in the current iteration.
- `B`: Batch size.
- `N`: Number of rays per batch.
- `n_alive`: Number of currently active rays.

### Algorithm Steps
1.  **Initialization**: Initialize `rays_alive[0]` to all `True` (all rays start alive). Initialize `rays_t[0]` to `near`. Initialize `weights_sum` and `depth` to 0. Initialize `step` (global step counter) to 0. Initialize `alive_counter` to $B \times N$.
2.  **Loop Condition**: While `alive_counter > 0` and `step < max_steps` (if a global max step limit is enforced):
    a.  **Calculate Alive Rays**: Determine `n_alive = alive_counter.item()`.
    b.  **Exit Check**: If `n_alive <= 0`, break the loop.
    c.  **Calculate Steps**: Compute `n_step = max(min(B * N // n_alive, 8), 1)`. This formula ensures that the total number of steps processed in this iteration ($n\_alive \times n\_step$) is approximately equal to the total number of rays ($B \times N$), capped at a maximum of 8 steps per ray to maintain stability and cache efficiency. If `n_alive` is small, `n_step` defaults to 1.
    d.  **Ray Marching**: Call `raymarching.march_rays` with:
        -   `n_alive`: Number of active rays.
        -   `n_step`: Steps per ray.
        -   `rays_alive[i % 2]`: Current alive mask.
        -   `rays_t[i % 2]`: Current ray distances.
        -   `rays_o`, `rays_d`: Ray origins and directions.
        -   `self.bound`, `self.density_grid`, `self.mean_density`: Scene data.
        -   `near`, `far`: Clipping planes.
        -   `128`: Block size or thread block dimension (implementation-specific constant).
        -   `perturb`: Perturbation flag.
        This function updates `rays_alive[(i+1) % 2]` and `rays_t[(i+1) % 2]` for the next iteration, marking rays as dead if they hit a surface or exceed bounds. It returns sampled positions `xyzs` and directions `dirs` for the active steps.
    e.  **Neural Network Query**: Compute densities (`sigmas`) and colors (`rgbs`) by passing `xyzs` and `dirs` through the neural network: `sigmas, rgbs = self(xyzs, dirs)`.
    f.  **Compositing**: Call `raymarching.composite_rays` with:
        -   `n_alive`, `n_step`: Ray counts.
        -   `rays_alive[i % 2]`, `rays_t[i % 2]`: Ray state.
        -   `sigmas`, `rgbs`: Network outputs.
        -   `deltas`: Step sizes (returned from marching or computed).
        -   `weights_sum`, `depth`: Accumulators.
        -   `image`: Current image buffer.
        This function updates `weights_sum`, `depth`, and `image` based on volume rendering integration.
    g.  **Update Step Counter**: `step += n_step`.
    h.  **Background Compositing**:
        -   If `alpha_premultiplied` is true: `image = image + (1 - weights_sum).unsqueeze(-1) * bg_color`.
        -   If `alpha_premultiplied` is false: `image /= (weights_sum + 1e-9).unsqueeze(-1)`. This normalizes the RGB values by the accumulated opacity. The `1e-9` prevents division by zero [S1, unofficial].
    i.  **Concatenate Weight Sum**: `image = torch.cat([image, weights_sum.unsqueeze(-1)], -1)`.
    j.  **Swap Buffers**: Swap `rays_alive` and `rays_t` buffers for the next iteration (toggle `i % 2`).
    k.  **Update Alive Counter**: Recalculate `alive_counter` based on the new `rays_alive` mask.
3.  **Final Reshape**: Reshape `image` to $[B, N, 4]$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ray_marching(rays_o, rays_d, near, far, bg_color=1.0, alpha_premultiplied=False, perturb=False):
    B, N, _ = rays_o.shape
    # Initialize buffers
    rays_alive = [torch.ones(B, N, dtype=torch.bool), torch.zeros(B, N, dtype=torch.bool)]
    rays_t = [torch.full((B, N), near, dtype=torch.float32), torch.zeros((B, N), dtype=torch.float32)]
    weights_sum = torch.zeros(B, N, dtype=torch.float32)
    depth = torch.zeros(B, N, dtype=torch.float32)
    image = torch.zeros(B, N, 3, dtype=torch.float32)
    
    alive_counter = B * N
    step = 0
    i = 0
    
    while alive_counter > 0:
        n_alive = alive_counter
        if n_alive <= 0:
            break
            
        # Calculate steps per ray
        n_step = max(min((B * N) // n_alive, 8), 1)
        
        # March rays
        # Returns: xyzs, dirs, deltas, new_alive_mask, new_t
        xyzs, dirs, deltas, new_alive_mask, new_t = raymarching.march_rays(
            n_alive, n_step, 
            rays_alive[i % 2], rays_t[i % 2],
            rays_o, rays_d, 
            bound=self.bound, density_grid=self.density_grid, 
            mean_density=self.mean_density, near=near, far=far, 
            block_size=128, perturb=perturb
        )
        
        # Query network
        sigmas, rgbs = self.network(xyzs, dirs)
        
        # Composite
        weights_sum, depth, image = raymarching.composite_rays(
            n_alive, n_step,
            rays_alive[i % 2], rays_t[i % 2],
            sigmas, rgbs, deltas, 
            weights_sum, depth, image
        )
        
        # Update step counter
        step += n_step
        
        # Background compositing and normalization
        if alpha_premultiplied:
            image = image + (1 - weights_sum).unsqueeze(-1) * bg_color
        else:
            # Avoid division by zero [S1, unofficial]
            image = image / (weights_sum + 1e-9).unsqueeze(-1)
            
        # Concatenate weight sum
        image = torch.cat([image, weights_sum.unsqueeze(-1)], dim=-1)
        
        # Swap buffers
        rays_alive[(i + 1) % 2] = new_alive_mask
        rays_t[(i + 1) % 2] = new_t
        
        # Update alive counter for next iteration
        alive_counter = new_alive_mask.sum().item()
        i += 1
        
    # Reshape output
    image = image.reshape(B, N, 4)
    return image
```

### Edge Cases and Pitfalls
-   **Division by Zero**: When `alpha_premultiplied` is false, the normalization step divides by `weights_sum`. If a ray has zero accumulated opacity (e.g., passed through empty space), this causes division by zero. The implementation adds `1e-9` to the denominator to avoid this [S1, unofficial].
-   **Zero Alive Rays**: The loop checks `n_alive <= 0` to exit early, preventing unnecessary computation.
-   **Step Count Calculation**: The formula `max(min(B * N // n_alive, 8), 1)` ensures that `n_step` is at least 1. If `n_alive` is very small, `n_step` becomes 1, processing rays one by one in terms of step batches, which may reduce efficiency but prevents excessive steps per ray.

## Subtopics

### Adaptive Ray Marching with Hash Grid Acceleration
This variant uses a hash grid (specifically a density bitfield) for acceleration instead of a dense grid. It calculates the number of samples per ray based on the scene scale and a step factor. For each alive ray, it marches along the ray, checking the hash grid for occupied voxels. If a voxel is occupied, it samples the neural network; otherwise, it skips to the next occupied voxel. The step size is calculated using an exponential step factor (`exp_step_factor`), which is `1/256` if the scene scale is greater than 0.5, otherwise 0 [S2, unofficial]. The hash grid uses morton codes for indexing, and the density bitfield is stored as a bitfield of uint8 values [S2, unofficial]. Parameters include `grid_size` (resolution of the hash grid), `cascades` (number of levels in the density bitfield), and `scale` (size of the scene bounding box) [S2, unofficial].

## Operating characteristics
-   **Time Complexity**: The complexity depends on the number of active rays and the number of steps per ray. In the worst case (all rays active), it processes $B \times N$ rays with up to 8 steps each per iteration. The number of iterations depends on the scene density and ray length.
-   **Memory Complexity**: Requires storage for ping-pong buffers (`rays_alive`, `rays_t`), neural network weights, and the acceleration structure (density grid or hash grid). The hash grid variant uses a bitfield for density, reducing memory usage compared to a dense grid.
-   **Performance**: The dynamic step count optimization improves throughput by batching rays, reducing kernel launch overhead and improving cache locality. The hash grid acceleration significantly speeds up empty space skipping.
-   **Strengths**: Efficient handling of sparse scenes, reduced computation on terminated rays, good hardware utilization.
-   **Limitations**: Performance may degrade if the number of active rays is very small (leading to small batch sizes) or very large (leading to many iterations). The fixed step cap (8) may limit accuracy in highly detailed regions.

## Background
This method is derived from implementations in neural radiance fields (NeRF) and instant-NGP, where adaptive ray marching is used to optimize rendering speed. The specific implementation with dynamic step count and hash grid acceleration is found in unofficial repositories such as `cv-master-ex` and `taichi-ngp-renderer` [S1, S2, unofficial].

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S2] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
