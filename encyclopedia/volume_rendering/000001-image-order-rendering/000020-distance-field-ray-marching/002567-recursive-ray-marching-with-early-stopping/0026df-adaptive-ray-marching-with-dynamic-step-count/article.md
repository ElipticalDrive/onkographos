# Adaptive Ray Marching with Dynamic Step Count

Also called: CUDA Ray Marching with Adaptive Step Count

## Overview
Adaptive Ray Marching with Dynamic Step Count is an iterative volume rendering algorithm that samples rays until they are occluded or a maximum step count is reached, using a density grid for acceleration [S1, unofficial]. It differs from its parent, [Recursive Ray Marching with Early Stopping](3546-recursive-ray-marching-with-early-stopping.md), by dynamically adjusting the number of steps per ray based on the count of currently active rays [S1, unofficial].

## How it works
The method operates in an iterative loop that determines the number of currently active rays and calculates a step batch size for them [S1, unofficial]. It proceeds by marching these rays, querying densities and colors, and compositing the results [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- `rays_o`: Ray origins, shape $[B, N, 3]$ [S1, unofficial].
- `rays_d`: Ray directions, shape $[B, N, 3]$ [S1, unofficial].
- `near`, `far`: Near and far clipping planes [S1, unofficial].
- `bound`: Bounding box parameters [S1, unofficial].
- `density_grid`: Acceleration structure (density grid) [S1, unofficial].
- `mean_density`: Mean density value for normalization or thresholding [S1, unofficial].
- `bg_color`: Background color, default $1.0$ [S1, unofficial].
- `perturb`: Boolean flag for ray perturbation [S1, unofficial].
- `alpha_premultiplied`: Boolean flag determining output normalization [S1, unofficial].
- `B`: Batch size [S1, unofficial].
- `N`: Number of rays per batch [S1, unofficial].

**Outputs:**
- `image`: Rendered image, shape $[B, N, 4]$ [S1, unofficial]. The first three channels are RGB, and the fourth channel is the accumulated weight sum (transparency mask) [S1, unofficial].

### Data Structures
- `rays_alive`: A ping-pong buffer array of two boolean tensors, shape $[2, B, N]$. `rays_alive[i % 2]` indicates which rays are still active in the current iteration $i$ [S1, unofficial].
- `rays_t`: A ping-pong buffer array of two float tensors, shape $[2, B, N]$. `rays_t[i % 2]` stores the current accumulated distance along the ray for active rays [S1, unofficial].
- `alive_counter`: An integer scalar tracking the total number of active rays across the batch [S1, unofficial].
- `weights_sum`: Accumulated weight (opacity) per ray, shape $[B, N]$ [S1, unofficial].
- `depth`: Accumulated depth per ray, shape $[B, N]$ [S1, unofficial].

### Parameters
- `n_step`: Number of steps to process per alive ray in the current iteration [S1, unofficial].
- `B`: Batch size [S1, unofficial].
- `N`: Number of rays per batch [S1, unofficial].
- `n_alive`: Number of currently active rays [S1, unofficial].

### Algorithm Steps
1. **Initialization**: Initialize `rays_alive[0]` to all `True` (all rays start alive). Initialize `rays_t[0]` to `near`. Initialize `weights_sum` and `depth` to $0$. Initialize `step` (global step counter) to $0$. Initialize `alive_counter` to $B \times N$ [S1, unofficial].
2. **Loop Condition**: While `alive_counter > 0` and `step < max_steps` (if a global max step limit is enforced):
   a. **Calculate Alive Rays**: Determine `n_alive = alive_counter.item()` [S1, unofficial].
   b. **Exit Check**: If `n_alive <= 0`, break the loop [S1, unofficial].
   c. **Calculate Steps**: Compute `n_step = max(min(B * N // n_alive, 8), 1)` [S1, unofficial].
   d. **Ray Marching**: Call `raymarching.march_rays` with `n_alive`, `n_step`, `rays_alive[i % 2]`, `rays_t[i % 2]`, `rays_o`, `rays_d`, `bound`, `density_grid`, `mean_density`, `near`, `far`, `128`, `perturb` [S1, unofficial]. This function updates `rays_alive[(i+1) % 2]` and `rays_t[(i+1) % 2]` for the next iteration, marking rays as dead if they hit a surface or exceed bounds. It returns sampled positions `xyzs` and directions `dirs` for the active steps [S1, unofficial].
   e. **Neural Network Query**: Compute densities (`sigmas`) and colors (`rgbs`) by passing `xyzs` and `dirs` through the neural network: `sigmas, rgbs = network_forward(xyzs, dirs)` [S1, unofficial].
   f. **Compositing**: Call `raymarching.composite_rays` with `n_alive`, `n_step`, `rays_alive[i % 2]`, `rays_t[i % 2]`, `sigmas`, `rgbs`, `deltas`, `weights_sum`, `depth`, `image` [S1, unofficial]. This function updates `weights_sum`, `depth`, and `image` based on volume rendering integration [S1, unofficial].
   g. **Update Step Counter**: `step += n_step` [S1, unofficial].
   h. **Background Compositing**: If `alpha_premultiplied` is true, `image = image + (1 - weights_sum).unsqueeze(-1) * bg_color` (default `bg_color = 1`) [S1, unofficial]. If `alpha_premultiplied` is false, `image /= (weights_sum + 1e-9).unsqueeze(-1)` [S1, unofficial].
   i. **Concatenate Weight Sum**: `image = torch.cat([image, weights_sum.unsqueeze(-1)], -1)` [S1, unofficial].
   j. **Swap Buffers**: Swap `rays_alive` and `rays_t` buffers for the next iteration (toggle `i % 2`) [S1, unofficial].
   k. **Update Alive Counter**: Recalculate `alive_counter` based on the new `rays_alive` mask [S1, unofficial].
3. **Final Reshape**: Reshape `image` to $[B, N, 4]$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ray_marching(rays_o, rays_d, near, far, bg_color=1.0, alpha_premultiplied=False, perturb=False, bound=None, density_grid=None, mean_density=None, B=1, N=1):
    # Initialize ping-pong buffers and accumulators
    rays_alive = [ [True] * (B * N), [False] * (B * N) ]
    rays_t = [ [near] * (B * N), [0.0] * (B * N) ]
    weights_sum = [0.0] * (B * N)
    depth = [0.0] * (B * N)
    image = [0.0] * (B * N * 3)
    
    alive_counter = B * N
    step = 0
    i = 0
    
    while alive_counter > 0:
        n_alive = alive_counter
        if n_alive <= 0:
            break
        
        n_step = max(min((B * N) // n_alive, 8), 1)
        
        # March rays: updates alive mask and t, returns sampled xyzs, dirs, deltas
        xyzs, dirs, deltas, new_alive_mask, new_t = march_rays(
            n_alive, n_step, rays_alive[i % 2], rays_t[i % 2],
            rays_o, rays_d, bound, density_grid, mean_density, near, far, 128, perturb
        )
        
        # Query neural network for densities and colors
        sigmas, rgbs = network_forward(xyzs, dirs)
        
        # Composite rays: updates weights_sum, depth, image
        weights_sum, depth, image = composite_rays(
            n_alive, n_step, rays_alive[i % 2], rays_t[i % 2],
            sigmas, rgbs, deltas, weights_sum, depth, image
        )
        
        step += n_step
        
        # Background compositing or normalization
        if alpha_premultiplied:
            for k in range(B * N):
                image[k*3] += (1.0 - weights_sum[k]) * bg_color
                image[k*3+1] += (1.0 - weights_sum[k]) * bg_color
                image[k*3+2] += (1.0 - weights_sum[k]) * bg_color
        else:
            for k in range(B * N):
                denom = weights_sum[k] + 1e-9
                image[k*3] /= denom
                image[k*3+1] /= denom
                image[k*3+2] /= denom
        
        # Concatenate weight sum to image channels
        for k in range(B * N):
            image.append(weights_sum[k])
        
        # Swap ping-pong buffers
        rays_alive[(i + 1) % 2] = new_alive_mask
        rays_t[(i + 1) % 2] = new_t
        
        # Update alive counter
        alive_counter = sum(1 for x in new_alive_mask if x)
        i += 1
        
    # Reshape output to [B, N, 4]
    image = image[:B * N * 4]
    return image

def march_rays(n_alive, n_step, rays_alive, rays_t, rays_o, rays_d, bound, density_grid, mean_density, near, far, block_size, perturb):
    # GPU kernel: marches rays, checks bounds/density, updates alive mask and t
    # Returns: xyzs, dirs, deltas, new_alive_mask, new_t
    return [], [], [], [], []

def composite_rays(n_alive, n_step, rays_alive, rays_t, sigmas, rgbs, deltas, weights_sum, depth, image):
    # GPU kernel: volume rendering integration, updates weights_sum, depth, image
    return weights_sum, depth, image

def network_forward(xyzs, dirs):
    # Neural network forward pass
    return [], []
```

### Edge Cases and Pitfalls
- **Division by Zero**: When `alpha_premultiplied` is false, the normalization step divides by `weights_sum`. If a ray has zero accumulated opacity (e.g., passed through empty space), this causes division by zero. The implementation adds `1e-9` to the denominator to avoid this [S1, unofficial].
- **Zero Alive Rays**: The loop checks `n_alive <= 0` to exit early, preventing unnecessary computation [S1, unofficial].
- **Step Count Calculation**: The formula `max(min(B * N // n_alive, 8), 1)` ensures that `n_step` is at least $1$. If `n_alive` is very small, `n_step` becomes $1$, processing rays one by one in terms of step batches, which may reduce efficiency but prevents excessive steps per ray [S1, unofficial].

## Subtopics

### Adaptive Ray Marching with Hash Grid Acceleration
This variant uses a hash grid (specifically a density bitfield) for acceleration instead of a dense grid [S2, unofficial]. It calculates the number of samples per ray based on the scene scale and a step factor [S2, unofficial]. For each alive ray, it marches along the ray, checking the hash grid for occupied voxels [S2, unofficial]. If a voxel is occupied, it samples the neural network; otherwise, it skips to the next occupied voxel [S2, unofficial]. The step size is calculated using an exponential step factor (`exp_step_factor`), which is $1/256$ if the scene scale is greater than $0.5$, otherwise $0$ [S2, unofficial]. The hash grid uses morton codes for indexing, and the density bitfield is stored as a bitfield of uint8 values [S2, unofficial]. Parameters include `grid_size` (resolution of the hash grid), `cascades` (number of levels in the density bitfield), and `scale` (size of the scene bounding box) [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Requires storage for ping-pong buffers (`rays_alive`, `rays_t`), neural network weights, and the acceleration structure (density grid or hash grid) [S1, unofficial][S2, unofficial].
- **Performance**: Not given in the sources.
- **Strengths**: Efficient handling of sparse scenes, reduced computation on terminated rays, good hardware utilization [S1, unofficial][S2, unofficial].
- **Limitations**: Performance may degrade if the number of active rays is very small (leading to small batch sizes) or very large (leading to many iterations). The fixed step cap ($8$) may limit accuracy in highly detailed regions [S1, unofficial].

## Background
This method is derived from implementations in neural radiance fields (NeRF) and instant-NGP, where adaptive ray marching is used to optimize rendering speed [S1, unofficial][S2, unofficial]. The specific implementation with dynamic step count and hash grid acceleration is found in unofficial repositories [S1, unofficial][S2, unofficial].

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S2] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
