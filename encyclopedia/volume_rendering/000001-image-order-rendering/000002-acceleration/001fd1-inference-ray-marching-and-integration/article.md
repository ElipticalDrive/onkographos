# Inference Ray Marching and Integration

Also called: Inference Ray Marching, Ray Marching for Inference, CUDA Ray Marching (Inference), Sparse Ray Marching (Inference), Ray Marching (CUDA Kernel)

## Overview
Inference Ray Marching and Integration is a GPU-accelerated pipeline for real-time volume rendering inference, designed to generate images from trained neural radiance fields (NeRFs) or sparse density grids without gradient computation [S1, S2, S3, S4, unofficial]. It solves the problem of efficiently traversing sparse volumetric data to sample points and integrate radiance, utilizing density bitfields or occupancy grids for acceleration and early termination to maximize throughput [S1, S2, S3, unofficial]. It differs from training-time marchers by disabling gradient flow, using fixed or pre-computed step sizes, and operating on pre-computed ray samples with strict early-termination logic to prioritize latency over optimization stability [S2, S3, unofficial].

## How it works
The method consists of two primary phases: **Ray Marching** (sampling) and **Integration** (compositing) [S2, S3, unofficial]. The marching phase generates sample points along rays, skipping empty space using an acceleration structure [S4, unofficial]. The integration phase evaluates the neural network at these points and accumulates color and depth [S3, unofficial].

### Inputs and Outputs
**Inputs:**
*   `rays_o`: Float array `[N, 3]`, ray origins for $N$ rays [S1, S2, S4, unofficial].
*   `rays_d`: Float array `[N, 3]`, ray directions for $N$ rays [S1, S2, S4, unofficial].
*   `near`: Float array `[N]`, near plane distances [S1, S2, S4, unofficial].
*   `far`: Float array `[N]`, far plane distances [S1, S2, S4, unofficial].
*   `density_bitfield`: Uint8 array `[CHHH // 8]`, hashed grid for density acceleration (Instant-NGP style) [S1, S4, unofficial].
*   `density_grid`: Float array `[H, H, H]`, pre-computed 3D density grid for occupancy pruning (alternative to bitfield) [S2, S3, unofficial].
*   `mean_density`: Float, average density value in the grid, used for step size estimation [S2, unofficial].
*   `bound`: Float, scalar bounding box size (scene bounded in $[-bound, bound]^3$) [S1, S2, S4, unofficial].
*   `dt_gamma`: Float, cone angle for exponential acceleration. If $>0$, accelerates marching but may reduce quality [S1, S4, unofficial]. Default $1/256$ [S1, unofficial].
*   `perturb`: Bool or Int. If true (or int seed), adds jitter to sample positions for anti-aliasing [S1, S2, S4, unofficial].
*   `max_steps`: Int, maximum number of sampled points along each ray. Default 1024 [S1, S4, unofficial].
*   `align`: Int, pad output size to multiple of `align` for memory alignment. Default -1 (disabled) [S1, S2, unofficial].
*   `density_thresh`: Float, threshold for density grid to be considered occupied. Default 10 [S1, S4, unofficial].

**Outputs:**
*   `xyzs`: Float array `[M, 3]`, 3D coordinates of sampled points [S2, S4, unofficial].
*   `dirs`: Float array `[M, 3]`, ray directions corresponding to each sampled point [S2, S4, unofficial].
*   `deltas`: Float array `[M, 2]`, two delta values per point: first for RGB contribution, second for depth contribution [S2, S4, unofficial].
*   `image`: Float array `[N, 3]`, final rendered RGB image [S2, unofficial].
*   `depth`: Float array `[N, 1]`, final rendered depth image [S2, unofficial].

### Data Structures
*   **Alive Ray Buffers:**
    *   `rays_alive`: Int array `[N]`, IDs of currently alive rays. Only the first `n_alive` entries are valid [S1, S2, S4, unofficial].
    *   `rays_t`: Float array `[N]`, current traversal position ($t$) along each ray [S1, S2, S4, unofficial].
*   **Output Tensors:**
    *   `xyzs`, `dirs`, `deltas`: Pre-allocated zero tensors of shape `[M, 3]`, `[M, 3]`, `[M, 2]` respectively, where $M$ is the total output size [S2, S4, unofficial].
*   **Rendering State (for Integration):**
    *   `rays_rgbd`: Float array `[N, 4]`, initialized to zeros, accumulates RGBA [S3, unofficial].
    *   `rays_T`: Float array `[N]`, initialized to ones, stores transmittance [S3, unofficial].
    *   `rays_cost`: Float array `[N]`, optional, initialized to zeros if cost rendering is enabled [S3, unofficial].

### Algorithm Steps

1.  **Initialization:**
    *   Ensure `rays_o` and `rays_d` are contiguous on the CUDA device [S1, S2, S4, unofficial].
    *   Reshape `rays_o` and `rays_d` to `[-1, 3]` [S2, unofficial].
    *   Initialize `rays_alive` with indices `0` to `n_alive - 1` [S2, unofficial].
    *   Initialize `rays_t` with `near` distances [S2, unofficial].
    *   Set `n_alive = N` (total rays) [S2, unofficial].
    *   If `perturb` is true, generate random noise `[n_alive]` for jittering [S1, unofficial].

2.  **Ray Marching Loop:**
    *   Loop while `n_alive > 0` and step count `< max_steps` [S2, S4, unofficial].
    *   **Step Size Calculation:**
        *   Calculate `n_step` per ray. A common heuristic is `n_step = max(min(B * N // n_alive, 8), 1)` where $B$ is a batch factor, or use a fixed cap [S2, unofficial].
        *   If `dt_gamma > 0`, apply exponential acceleration: step size decreases as rays progress, proportional to $dt\_gamma \times t$ [S1, S4, unofficial].
    *   **Output Size Calculation:**
        *   Calculate total output size $M = n\_alive \times n\_step$ [S1, S2, S4, unofficial].
        *   If `align > 0`, pad $M$ to the next multiple of `align`: $M \leftarrow M + (align - (M \% align)) \% align$ [S1, S2, unofficial].
    *   **CUDA Kernel Invocation:**
        *   Call the backend kernel `_backend.march_rays` (or equivalent) with parameters: `n_alive`, `n_step`, `rays_alive`, `rays_t`, `rays_o`, `rays_d`, `bound`, `density_bitfield` (or `density_grid`), `C` (hash grid channels), `H` (grid resolution), `near`, `far`, `align`, `perturb`, `dt_gamma`, `max_steps` [S1, S2, S4, unofficial].
        *   The kernel populates `xyzs`, `dirs`, and `deltas` for the active rays [S2, S4, unofficial].
        *   Inside the kernel:
            *   For each alive ray, compute sample positions $t_i = t_{current} + \Delta t$ [S2, unofficial].
            *   Query the density grid/bitfield at $xyz_i$ [S2, unofficial].
            *   If density $< density\_thresh$, skip the point (empty space) [S1, S2, S4, unofficial].
            *   If occupied, write $xyz_i$, $dir$, and $\Delta t$ to output buffers [S2, S4, unofficial].
            *   Update `rays_t` for the next step [S2, unofficial].
    *   **Compaction:**
        *   After the kernel, compact the list of alive rays. Rays that have reached `far` or exceeded `max_steps` are removed from `rays_alive` [S2, unofficial].
        *   Update `n_alive` to the number of remaining active rays [S2, unofficial].

3.  **Integration (Compositing):**
    *   Initialize `rays_rgbd` to zeros, `rays_T` to ones [S3, unofficial].
    *   Loop through the sampled points generated by the marching phase [S3, unofficial].
    *   For each sample point $i$ along a ray:
        *   Evaluate the NeRF network at $xyz_i$ with `stop_gradient` enabled to prevent backpropagation [S3, unofficial].
        *   Get density $\sigma_i$ and color $c_i$ [S2, unofficial].
        *   Compute weight $w_i = \sigma_i \Delta t_i \exp(-\sum_{j<i} \sigma_j \Delta t_j)$ [S2, S3, unofficial].
        *   Update transmittance: $T_{new} = T_{old} \times (1 - w_i)$ [S3, unofficial].
        *   Accumulate color and depth based on weights [S3, unofficial].
    *   Count terminated rays and update `n_alive` for the next batch if processing in chunks [S3, unofficial].

4.  **Finalization:**
    *   Split `rays_rgbd` into RGB and depth buffers [S3, unofficial].
    *   Convert to final image formats (e.g., uint8) [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inference_ray_marching_and_integration(rays_o, rays_d, near, far, density_bitfield, density_grid, mean_density, bound, dt_gamma, perturb, max_steps, align, density_thresh):
    """
    Self-contained implementation of Inference Ray Marching and Integration.
    """
    import numpy as np
    
    N = rays_o.shape[0]
    rays_alive = np.arange(N, dtype=np.int32)
    rays_t = near.copy()
    n_alive = N
    
    xyzs_all = []
    dirs_all = []
    deltas_all = []
    
    step_count = 0
    while n_alive > 0 and step_count < max_steps:
        n_step = max(min(8, 1024 // n_alive), 1)
        M = n_alive * n_step
        if align > 0:
            M += align - (M % align)
            
        xyzs = np.zeros((M, 3), dtype=np.float32)
        dirs = np.zeros((M, 3), dtype=np.float32)
        deltas = np.zeros((M, 2), dtype=np.float32)
        
        write_idx = 0
        dead_mask = np.zeros(n_alive, dtype=bool)
        
        for i in range(n_alive):
            ray_idx = rays_alive[i]
            t_curr = rays_t[ray_idx]
            
            for s in range(n_step):
                step = dt_gamma * t_curr if dt_gamma > 0 else (far[ray_idx] - t_curr) / n_step
                t_next = min(t_curr + step, far[ray_idx])
                
                if t_next >= far[ray_idx]:
                    dead_mask[i] = True
                    
                xyz = rays_o[ray_idx] + t_next * rays_d[ray_idx]
                
                if np.any(np.abs(xyz) > bound):
                    t_curr = t_next
                    continue
                    
                sigma = query_density(xyz, density_bitfield, density_grid, bound)
                
                if sigma < density_thresh:
                    t_curr = t_next
                    continue
                    
                xyzs[write_idx] = xyz
                dirs[write_idx] = rays_d[ray_idx]
                deltas[write_idx, 0] = t_next - t_curr
                deltas[write_idx, 1] = t_next - t_curr
                write_idx += 1
                t_curr = t_next
                
            rays_t[ray_idx] = t_curr
            
        rays_alive = rays_alive[~dead_mask]
        n_alive = len(rays_alive)
        
        xyzs_all.append(xyzs[:write_idx])
        dirs_all.append(dirs[:write_idx])
        deltas_all.append(deltas[:write_idx])
        step_count += 1
        
    if xyzs_all:
        xyzs_all = np.concatenate(xyzs_all, axis=0)
        dirs_all = np.concatenate(dirs_all, axis=0)
        deltas_all = np.concatenate(deltas_all, axis=0)
    else:
        xyzs_all = np.empty((0, 3), dtype=np.float32)
        dirs_all = np.empty((0, 3), dtype=np.float32)
        deltas_all = np.empty((0, 2), dtype=np.float32)
        
    return integrate_rays_inference(rays_o, rays_d, xyzs_all, dirs_all, deltas_all, near, far, rays_alive, rays_t)

def query_density(xyz, density_bitfield, density_grid, bound):
    # [G] Standard hash grid or trilinear interpolation lookup.
    # Returns sigma value at xyz.
    return 0.0 # Placeholder for grid query logic

def integrate_rays_inference(rays_o, rays_d, xyzs, dirs, deltas, near, far, rays_alive, rays_t):
    N = rays_o.shape[0]
    rays_rgbd = np.zeros((N, 4), dtype=np.float32)
    rays_T = np.ones(N, dtype=np.float32)
    rays_depth = np.zeros(N, dtype=np.float32)
    
    # [G] Group samples by ray index. In CUDA, this is done via offsets or sorting.
    # Here we assume a simple mapping for demonstration.
    ray_sample_indices = np.argsort(np.concatenate([np.full(len(xyzs), i) for i in range(N)])) # Simplified grouping
    
    for i in range(N):
        T = rays_T[i]
        color = rays_rgbd[i, :3].copy()
        depth_acc = rays_depth[i]
        
        # Get samples for ray i
        # In practice, iterate through xyzs/deltas assigned to ray i
        # For self-contained pseudocode, we simulate the loop
        for k in range(len(xyzs)):
            # Check if sample belongs to ray i (simplified)
            pass 
            
        # Standard front-to-back compositing loop
        # alpha = 1.0 - np.exp(-sigma * delta)
        # w = alpha * T
        # color += rgb * w
        # depth_acc += xyz.dot(dir) * w
        # T *= (1.0 - alpha)
        pass
        
    return rays_rgbd[:, :3], rays_depth
```

### Parameters and Typical Values
*   `dt_gamma`: $1/256$ [S1, unofficial].
*   `max_steps`: 1024 [S1, S4, unofficial].
*   `density_thresh`: 10 [S1, S4, unofficial].
*   `bound`: 1.0 [S4, unofficial].
*   `perturb`: True [S3, unofficial].
*   `align`: -1 (disabled) [S1, S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Use `float32` for ray positions and densities to avoid accumulation errors [S2, S4, unofficial].
*   **Empty Space:** If the density grid is sparse, ensure the acceleration structure (bitfield/grid) correctly identifies empty regions to skip sampling [S1, S2, unofficial].
*   **Boundary Conditions:** Rays starting outside the bounding box or with `near > far` should be handled gracefully (e.g., skipped or rendered as background) [S1, S4, unofficial].
*   **Memory Alignment:** Failing to align output sizes can cause CUDA kernel launch failures or performance degradation [S1, S2, unofficial].

## Subtopics

### [Inference Ray Compositing](3790-inference-ray-compositing.md)
The final rendering stage of the Inference Ray Marching and Integration pipeline, responsible for aggregating per-step volumetric properties (density, color, depth, ambient light, uncertainty) into final pixel buffers. It differs from training-time compositing by operating on pre-computed ray samples with strict early-termination logic to maximize throughput during real-time inference. It serves as the bridge between the sparse ray marching results and the final image output, handling multiple output channels (RGB, depth, ambient, uncertainty) in parallel.

### [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md)
A GPU-accelerated kernel designed to sample points along rays for the training phase of neural volume rendering, specifically within the Instant Neural Graphics Primitives (NGP) framework. It solves the problem of efficiently traversing a sparse, hashed density grid to identify occupied regions while minimizing global memory traffic. It distinguishes itself from inference marchers by supporting adaptive step sizes and perturbation for gradient-based optimization, and from static allocators by dynamically managing sample buffers based on occupancy thresholds.

### [Chunked Volume Rendering](4310-chunked-volume-rendering.md)
A memory-management technique for image-order volume rendering that processes rays in fixed-size batches (chunks) to bound GPU memory usage during inference. It solves the problem of out-of-memory errors when rendering high-resolution images or large ray bundles by decoupling the total ray count from the per-forward-pass memory footprint. It differs from [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md), which optimizes sampling density during training, and [Inference Ray Compositing](3790-inference-ray-compositing.md), which focuses on the compositing logic itself; Chunked Volume Rendering specifically addresses the *batching* of the entire ray processing pipeline.

### BRDF Evaluation Kernel
A CUDA kernel that evaluates a BRDF for a batch of rays, taking RGBA, position, gradient, and ray direction inputs to produce output RGBA. It loops over batch indices, fetches inputs, calls `brdf.eval`, and writes the result [S5, unofficial].

### [Inference Pipeline](5924-inference-pipeline.md)
Renders images from trained dense point cloud and new camera poses without optimization. It takes `DensePointCloudWithExtraFeatures` and `NewCameraPose` as input and uses a rasterizer to output an Image.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Real-time inference (30+ FPS) is achievable on modern GPUs (e.g., RTX 3090) for resolutions up to 1080p with optimized kernels [S3, S4, unofficial].
*   **Scaling:** Performance scales linearly with resolution and ray count, but is bounded by memory bandwidth and cache efficiency [S2, S4, unofficial].
*   **Strengths:** High throughput for inference, efficient memory usage with sparse structures, real-time capability [S1, S2, S3, unofficial].
*   **Limitations:** Quality may degrade with aggressive early termination or large `dt_gamma` [S1, S4, unofficial]. Not suitable for training due to lack of gradient support [S2, S3, unofficial].
*   **When to Use:** Real-time rendering, interactive applications, inference on large scenes [S1, S3, S4, unofficial].
*   **When Not to Use:** Training neural networks, high-quality offline rendering requiring many samples per pixel [S2, S3, unofficial].

## Background
Not given in the sources.

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S4] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S5] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S6] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
