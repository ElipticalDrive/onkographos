# Slab-based Raycasting

Also called: Slab-based Raycaster

## Overview
Slab-based raycasting is a CUDA GPU optimization for direct volume rendering that caches sampled voxels in shared memory as "slabs" to improve performance when volume data is accessed multiple times, such as for gradient calculation in Phong lighting [S1, unofficial]. It sets itself apart from basic raycasting by trading increased shared memory overhead and reduced early ray termination granularity for cache hits on repeated voxel access [S1, unofficial]. Unlike hardware texture caches, which benefit from large viewports where adjacent rays hit the same voxels, the slab cache is most efficient when the data set resolution is high compared to the viewport size [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset (e.g., 16-bit or 8-bit), ray origin and direction per pixel, thread block configuration ($bs_x, bs_y$), slab depth ($sd$).
*   **Output:** Rendered image with lighting (e.g., Phong) applied.

**Data Structures:**
*   **Shared Memory Cache:** A contiguous block of shared memory storing $bs_x \times bs_y \times sd$ sampled voxels [S1, unofficial].
*   **Voxel Layout:** The cache stores the voxel values (and potentially gradients) for the slab. The exact bit-packing depends on the data type (e.g., 16-bit integers for `vmhead`, 8-bit for `engine`) [S1, unofficial].

**Parameters:**
*   $bs_x, bs_y$: Thread block dimensions.
*   $sd$: Slab depth (number of voxels along the ray stored in the cache).
*   $sd_{opt}$: Optimal slab depth, which depends on the specific dataset [S1, unofficial].

**Algorithm Steps:**
1.  **Block Initialization:** Each thread in the block determines its ray parameters. The block calculates the spatial extent of the slab it needs to fetch.
2.  **Slab Fetch:** The threads cooperatively fetch the volume data corresponding to the slab depth $sd$ along their rays into the shared memory cache. This involves calculating global memory addresses for the start of the slab and fetching $bs_x \times bs_y \times sd$ voxels [S1, unofficial].
3.  **Border Handling:** Additional border voxels may be fetched to handle gradient calculations at the edges of the slab, introducing overhead [S1, unofficial].
4.  **Synchronization:** All threads in the block synchronize (`__syncthreads()`) to ensure the slab is fully loaded into shared memory before any thread begins processing [S1, unofficial].
5.  **Ray Casting Loop:** Threads iterate through the slab depth $sd$. For each step:
    *   Read voxel value and gradient from shared memory.
    *   Compute lighting (e.g., Phong) using the gradient.
    *   Perform volume integration (compositing).
    *   Check for early ray termination. Note: Termination can only occur at the end of a complete slab, not after individual rays within the slab, reducing efficiency for solid data [S1, unofficial].
6.  **Next Slab:** If the ray has not terminated and more volume remains, the block fetches the next slab into shared memory and repeats Step 3-5.
7.  **Write Output:** Write the final accumulated color and opacity to global memory.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shared memory declaration
__shared__ float shared_voxels[BS_X * BS_Y * SD];
__shared__ float shared_gradients[BS_X * BS_Y * SD * 3]; # Assuming 3-component gradients

def slab_based_raycast(ray_origin, ray_direction, volume_data, bs_x, bs_y, sd):
    # 1. Calculate slab bounds
    # Determine the range of t values for the current slab
    slab_start_t = current_slab_index * sd * ray_step_size
    slab_end_t = (current_slab_index + 1) * sd * ray_step_size
    
    # 2. Fetch slab into shared memory
    # Each thread fetches its portion of the slab
    tid = threadIdx.x + threadIdx.y * bs_x
    num_threads = bs_x * bs_y
    
    # Cooperative fetch logic
    # Threads iterate to fill shared_voxels and shared_gradients
    for i in range(0, sd):
        # Calculate global memory index for the voxel at current step
        global_idx = get_volume_index(ray_origin + ray_direction * (slab_start_t + i * ray_step_size))
        
        # Fetch voxel value
        shared_voxels[tid * sd + i] = volume_data[global_idx]
        
        # Fetch gradient for lighting (if using Phong)
        # Note: Gradient calculation may require neighbor fetches, handled via border logic [S1, unofficial]
        shared_gradients[(tid * sd + i) * 3 : (tid * sd + i) * 3 + 3] = compute_gradient(volume_data, global_idx)
        
        __syncthreads() # Ensure all threads have fetched their part before next step? 
                        # Actually, usually fetch is done in parallel, then sync. 
                        # Simplified: Fetch all, then sync.
    
    __syncthreads() # Ensure entire slab is in shared memory
    
    # 3. Process slab
    for i in range(0, sd):
        voxel_val = shared_voxels[tid * sd + i]
        grad = shared_gradients[(tid * sd + i) * 3 : (tid * sd + i) * 3 + 3]
        
        # Lighting calculation
        color = apply_phong_lighting(voxel_val, grad)
        
        # Compositing
        alpha = transfer_function(voxel_val)
        out_color = blend(out_color, color, alpha)
        out_alpha = blend(out_alpha, alpha)
        
        # Early termination check
        if out_alpha >= TERMINATION_THRESHOLD:
            break
            
    # 4. Increment slab index and repeat if needed
    current_slab_index += 1
```

**Edge Cases and Pitfalls:**
*   **Overhead:** For basic raycasting (no lighting/gradient), caching adds 23-67% overhead because each voxel is accessed only once [S1, unofficial].
*   **Early Termination:** Early ray termination is less efficient because only complete slabs can be terminated, not individual rays. This leads to performance decreases with solid/non-transparent data [S1, unofficial].
*   **Shared Memory Limits:** The cache size is $bs_x \times bs_y \times sd$. For optimal configurations, this can reach 15,360 bytes, close to the 16 kB maximum on older GPUs (e.g., GTX 280), limiting occupancy to one thread block per multiprocessor [S1, unofficial].
*   **Gradient Accuracy:** Gradients computed from slab data may be less exact than those from direct global memory access [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of slabs traversed. Each slab fetch is $O(bs_x \times bs_y \times sd)$, and processing is $O(sd)$. Overhead is significant for basic raycasting (23-67% slower) but beneficial for Phong lighting (12-78% faster) [S1, unofficial].
*   **Memory Complexity:** Shared memory usage is $bs_x \times bs_y \times sd$ voxels. For `vmhead`, this is 15,360 bytes; for `engine`, up to 7,936 bytes [S1, unofficial].
*   **Measured Performance:**
    *   **GTX 280, `vmhead` (512²×294, 16-bit), Phong, 512² viewport:** 67.9 fps (slab) vs 40.7 fps (basic), +78.2% speedup with $bs=16\times30, sd=16$ [S1, unofficial].
    *   **GTX 280, `engine` (256²×128, 8-bit), Phong, 512² viewport:** 77.1 fps (slab) vs 77.6 fps (basic), -0.6% change with $bs=8\times16$ [S1, unofficial].
*   **Scaling:** Speedup decreases with increasing viewport size. Larger viewports increase hardware texture cache hit rates due to spatial locality, reducing the relative benefit of the slab cache [S1, unofficial].
*   **Strengths:** Significant speedup for large, semi-transparent datasets with high resolution relative to viewport size, especially when lighting calculations require multiple voxel accesses [S1, unofficial].
*   **Limitations:** Poor performance for small, solid datasets or basic raycasting due to overhead and inefficient early termination. Limited by shared memory size, potentially reducing occupancy [S1, unofficial].
*   **When to Use:** Direct volume rendering of semi-transparent, high-resolution data with Phong lighting on GPUs with sufficient shared memory.
*   **When Not to Use:** Basic raycasting, small solid datasets, or when early ray termination is critical for performance [S1, unofficial].

## Background
Slab-based raycasting was developed as part of CUDA-based direct volume rendering techniques, specifically tested on the GeForce GTX 280 [S1, unofficial]. It was designed to address the inefficiency of repeated volume data access for lighting calculations (e.g., gradients for Phong shading) by leveraging shared memory [S1, unofficial]. The method is particularly relevant for medical imaging applications, where data sets typically have high resolution and are semi-transparent [S1, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28432/, fetched 2026-10-27 (unofficial)
