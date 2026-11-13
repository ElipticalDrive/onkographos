# AABB Ray Intersection

Also called: Batched AABB Ray Intersection, Ray-AABB Intersection (CUDA), Ray-AABB Intersection Kernel

## Overview
AABB Ray Intersection computes the intersection interval $[t_{min}, t_{max}]$ and hit status for a batch of rays against a batch of Axis-Aligned Bounding Boxes (AABBs). It solves the problem of efficiently determining which volumetric primitives (voxels, grids) a ray traverses in volume rendering pipelines. Unlike the scalar Slab/Min-Max method [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md), this method is optimized for GPU execution, utilizing parallelization over the product of rays and AABBs and handling batching constraints to prevent out-of-memory errors [S1, S3, S4, unofficial].

## How it works
The method operates on tensor inputs representing ray origins, directions, and AABB definitions. It computes intersection distances by evaluating the ray against the six planes defining the AABB (min/max for x, y, z axes).

### Inputs and Outputs
**Inputs:**
*   `rays_o`: Tensor of shape `[n_rays, 3]` containing ray origins $(O_x, O_y, O_z)$ [S2, S4, unofficial].
*   `rays_d`: Tensor of shape `[n_rays, 3]` containing ray directions $(D_x, D_y, D_z)$ [S2, S4, unofficial].
*   `aabbs`: Tensor of shape `[n_aabbs, 6]` containing AABB bounds. Each AABB is defined as $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ [S2, S4, unofficial].

**Outputs:**
*   `t_mins`: Tensor of shape `[n_rays, n_aabbs]` containing the entry distance $t_{min}$ for each ray-AABB pair [S4, unofficial].
*   `t_maxs`: Tensor of shape `[n_rays, n_aabbs]` containing the exit distance $t_{max}$ for each ray-AABB pair [S4, unofficial].
*   `hits`: Tensor of shape `[n_rays, n_aabbs]` (boolean) indicating if the ray intersects the AABB [S4, unofficial].

### Data Structures
The implementation typically uses contiguous GPU tensors (e.g., PyTorch CUDA tensors). For large batches, a chunking strategy is employed to manage memory usage [S3, unofficial].

### Algorithm Steps
The core intersection logic for a single ray and single AABB is parallelized across the grid. The batched implementation handles memory constraints by chunking rays [S3, unofficial].

1.  **Batching Setup (if required):**
    *   Calculate a chunk size $G = \min(2048, \lfloor 2 \times 10^9 / \text{points.numel()} \rfloor)$ to limit memory usage to approximately 2GB [S3, unofficial].
    *   Determine the number of chunks $K = \lceil N / G \rceil$ and padded size $H = K \times G$, where $N$ is the total number of rays [S3, unofficial].
    *   If $H > N$, pad `ray_start` and `ray_dir` by repeating the first $H-N$ rays [S3, unofficial].
    *   Reshape inputs to `(S * G, K, 3)` where $S$ is the spatial batch size (e.g., number of samples per ray) [S3, unofficial].

2.  **Parallel Kernel Execution:**
    *   Compute thread ID: `tid = blockIdx.x * blockDim.x + threadIdx.x` [S1, S4, unofficial].
    *   Loop while `tid < n_rays * n_aabbs` with stride `blockDim.x * gridDim.x` [S1, S4, unofficial].
    *   Calculate indices:
        $$ \text{ray\_id} = \lfloor \text{tid} / n_{\text{aabbs}} \rfloor $$
        $$ \text{aabb\_id} = \text{tid} \% n_{\text{aabbs}} $$
        [S1, S4, unofficial]

3.  **Intersection Calculation (Slab Method):**
    *   Compute direction fractions to avoid division by zero. Add a small epsilon $\epsilon = 10^{-6}$ to directions [S2, unofficial]:
        $$ \text{dir\_fraction} = \frac{1.0}{\text{rays\_d} + \epsilon} $$
        [S2, unofficial]
    *   For each axis $i \in \{x, y, z\}$, compute intersection distances with the min and max planes:
        $$ t_1 = (aabb_{min, i} - rays_{o, i}) \times \text{dir\_fraction}_i $$
        $$ t_2 = (aabb_{max, i} - rays_{o, i}) \times \text{dir\_fraction}_i $$
        [S2, unofficial]
        *(Note: Indices 0, 1, 2 correspond to x, y, z respectively in the tensor layout)*.

    *   Compute the entry distance $t_{min}$ (nears) as the maximum of the minimums of each axis pair:
        $$ t_{min} = \max(\min(t_1, t_2), \min(t_3, t_4), \min(t_5, t_6)) $$
        [S2, unofficial]

    *   Compute the exit distance $t_{max}$ (fars) as the minimum of the maximums of each axis pair:
        $$ t_{max} = \min(\max(t_1, t_2), \max(t_3, t_4), \max(t_5, t_6)) $$
        [S2, unofficial]

4.  **Hit Determination and Clamping:**
    *   Determine hit status: `hit = (t_max > t_min)` [S1, S4, unofficial].
    *   Clamp $t_{min}$ to the near plane (e.g., 0.0 during inference) [S2, unofficial]:
        $$ t_{min} = \max(t_{min}, \text{near\_plane}) $$
        [S2, unofficial]
    *   Ensure $t_{max} > t_{min}$ by at least $\epsilon = 10^{-6}$ to handle precision issues:
        $$ t_{max} = \max(t_{max}, t_{min} + 10^{-6}) $$
        [S2, unofficial]
    *   If `hit` is false, set $t_{min} = t_{max} = \text{miss\_value}$ (e.g., infinity) [S1, S4, unofficial].

5.  **Post-Processing:**
    *   Reshape outputs to `(S, H, -1)` [S3, unofficial].
    *   If padding was applied, truncate outputs to the original $N$ rays [S3, unofficial].
    *   Mark outputs as non-differentiable if gradients are not required for the intersection test itself [S3, unofficial].

### Pseudocode (CUDA Kernel Logic)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the device kernel logic, assuming inputs are flattened or indexed appropriately.
# This represents the logic inside the CUDA kernel called by the batched wrapper.

def ray_aabb_intersect_kernel(rays_o, rays_d, aabbs, t_mins, t_maxs, hits, n_rays, n_aabbs):
    # Thread indexing
    tid = (blockIdx.x * blockDim.x) + threadIdx.x
    stride = blockDim.x * gridDim.x
    
    # Total pairs
    total_pairs = n_rays * n_aabbs
    
    while tid < total_pairs:
        # Calculate ray and AABB indices
        ray_id = tid // n_aabbs
        aabb_id = tid % n_aabbs
        
        # Extract ray components
        ox = rays_o[ray_id * 3 + 0]
        oy = rays_o[ray_id * 3 + 1]
        oz = rays_o[ray_id * 3 + 2]
        
        dx = rays_d[ray_id * 3 + 0]
        dy = rays_d[ray_id * 3 + 1]
        dz = rays_d[ray_id * 3 + 2]
        
        # Extract AABB components
        xmin = aabbs[aabb_id * 6 + 0]
        ymin = aabbs[aabb_id * 6 + 1]
        zmin = aabbs[aabb_id * 6 + 2]
        xmax = aabbs[aabb_id * 6 + 3]
        ymax = aabbs[aabb_id * 6 + 4]
        zmax = aabbs[aabb_id * 6 + 5]
        
        # Compute direction fractions with epsilon to avoid division by zero
        eps = 1e-6
        inv_dx = 1.0 / (dx + eps)
        inv_dy = 1.0 / (dy + eps)
        inv_dz = 1.0 / (dz + eps)
        
        # Compute t values for x, y, z axes
        # t1, t2 for x
        t1 = (xmin - ox) * inv_dx
        t2 = (xmax - ox) * inv_dx
        
        # t3, t4 for y
        t3 = (ymin - oy) * inv_dy
        t4 = (ymax - oy) * inv_dy
        
        # t5, t6 for z
        t5 = (zmin - oz) * inv_dz
        t6 = (zmax - oz) * inv_dz
        
        # Compute t_min (entry)
        t_min = max(min(t1, t2), min(t3, t4), min(t5, t6))
        
        # Compute t_max (exit)
        t_max = min(max(t1, t2), max(t3, t4), max(t5, t6))
        
        # Hit test
        hit = (t_max > t_min)
        
        # Clamping and miss handling
        near_plane = 0.0 # Typical inference value
        if hit:
            t_min = max(t_min, near_plane)
            t_max = max(t_max, t_min + eps)
        else:
            t_min = float('inf')
            t_max = float('inf')
            hit = False
            
        # Store results
        t_mins[tid] = t_min
        t_maxs[tid] = t_max
        hits[tid] = hit
        
        tid += stride
```

### Parameters
*   `near_plane`: Minimum valid intersection distance. Typically 0.0 during inference, configurable during training [S2, unofficial].
*   `miss_value`: Value assigned to $t_{min}$ and $t_{max}$ when no intersection occurs. Typically infinity [S1, S4, unofficial].
*   `epsilon`: Small constant added to direction components to prevent division by zero. Value $10^{-6}$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Zero Direction Components:** Division by zero occurs if a ray direction component is exactly 0. The method handles this by adding $\epsilon$ to the direction before inversion [S2, unofficial].
*   **Out-of-Memory:** Processing all ray-AABB pairs simultaneously can exceed GPU memory limits. Mitigated by chunking rays into batches of size $G$ [S3, unofficial].
*   **Precision Issues:** Floating-point inaccuracies can cause $t_{max} < t_{min}$ for grazing rays. The method enforces $t_{max} \ge t_{min} + \epsilon$ [S2, unofficial].

## Subtopics

### Ray-AABB Intersection (PyTorch)
A pure PyTorch implementation of ray-AABB intersection. It is functionally identical to the CUDA version but slower due to the lack of specialized kernel optimization [S1, unofficial]. It uses broadcasting to compute intersection distances for all planes simultaneously: $t_1 = (aabb_{min} - rays_o) / rays_d$ and $t_2 = (aabb_{max} - rays_o) / rays_d$. Hits are determined by $t_{max} > t_{min}$ and $t_{max} > 0$, with results clipped to $[near\_plane, far\_plane]$ and set to `miss_value` on failure [S1, unofficial].

### Branchless Ray/AABB Intersection with NaN Handling
A variant designed to handle NaN values in ray directions or origins gracefully, often using branchless logic to maintain performance on GPUs [Branchless Ray/AABB Intersection with NaN Handling](3297-branchless-ray-aabb-intersection-with-nan-handling.md).

### AABB Intersection via Face Traversal
An alternative approach that traverses the grid by moving from one voxel face to the next, rather than computing explicit intersection distances for all six planes. This is often used for sparse voxel traversal [AABB Intersection via Face Traversal](13498-aabb-intersection-via-face-traversal.md).

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \times N_{aabbs})$ for the intersection computation. The chunking strategy adds overhead for data reshaping and padding but does not change the asymptotic complexity [S3, unofficial].
*   **Memory Complexity:** $O(N_{rays} \times N_{aabbs})$ for the output tensors. The chunking strategy reduces peak memory usage to $O(G \times N_{aabbs})$ where $G \le 2048$ [S3, unofficial].
*   **Performance:** The CUDA implementation is significantly faster than the pure PyTorch version due to parallel execution and optimized memory access patterns [S1, unofficial].
*   **Strengths:** High throughput for large batches; handles batching constraints effectively; robust to zero-direction components [S1, S2, S3, unofficial].
*   **Limitations:** Requires significant memory for large numbers of AABBs; chunking introduces CPU-GPU synchronization overhead if not managed carefully [S3, unofficial].

## Background
The method is widely used in modern volume rendering and NeRF (Neural Radiance Fields) implementations. It appears in libraries such as `nerfacc` and `sdfstudio` [S2, unofficial], and in research codebases like `taylor-nerf` [S1, unofficial] and `NSVF` [S3, unofficial]. The batching hack to prevent out-of-memory errors was introduced in `NSVF` [S3, unofficial]. The CUDA kernel implementation is standard in GPU-accelerated rendering pipelines [S4, unofficial].

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S3] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S4] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
