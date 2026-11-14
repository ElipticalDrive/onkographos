# Thread-efficient Partial Rendering

Also called: Multi-view Tiled Partial Rendering, Subsampled Tile Rendering

## Overview
Thread-efficient partial rendering is a GPU rasterization optimization for 3D Gaussian Splatting (3DGS) designed to reduce the computational overhead of multi-view training [S1, unofficial]. It solves the problem of low GPU utilization caused by naive binary masking in tiled rendering, where idle threads wait for active threads to finish [S1, unofficial]. Unlike naive multi-view training which renders all pixels for all views, or naive partial rendering with masks which leaves most threads in a block idle, this method maintains the same number of training pixels as single-view training while maximizing parallelization by assigning multiple GPU blocks to a single tile [S1, unofficial].

## How it works
The method modifies the standard tiled rasterization loop by decoupling the number of threads per block from the tile size, allowing multiple blocks to cooperatively render a single tile for multiple viewpoints.

**Inputs and Outputs:**
*   **Inputs:**
    *   $K$: The number of participating viewpoints [S1, unofficial].
    *   $N$: The number of pixels (and standard threads) in a tile [S1, unofficial].
    *   $D_I$: A sorted array of Gaussian primitives projected onto the image plane of viewpoint $I$ [S1, unofficial].
    *   $P_A$: An array mapping thread indices to specific pixel coordinates to render [S1, unofficial].
    *   $A$: An index array defining which pixels are subsampled for each viewpoint [S1, unofficial].
*   **Outputs:**
    *   Final color and opacity values written to pixel locations $P_A[k,i]$ [S1, unofficial].

**Data Structures:**
*   **Shared Memory (`Shmem_k`):** An array of size $N/K$ shared across all threads $i$ within block $k$ [S1, unofficial].
*   **Validity Flag (`isValid_{k,i}`):** A boolean flag for each thread indicating if its assigned pixel is within image bounds [S1, unofficial].
*   **Intermediate Value (`V_{k,i}`):** A temporal variable storing the intermediate result of alpha blending for thread $i$ in block $k$ [S1, unofficial].

**Parameters:**
*   $i$: Index of a thread in a tile [S1, unofficial].
*   $k$: Index of a GPU block, ranging from $1$ to $K$ [S1, unofficial].
*   $N/K$: The number of threads per block, which must equal the number of subsampled pixels for the assigned viewpoint [S1, unofficial].

**Algorithm Steps:**
1.  **Subsampling:** For each viewpoint, subsample the pixels to be rendered within a tile. Pass these subsampled pixel indices as an index array to the rasterizer [S1, unofficial].
2.  **Block Assignment:** Assign $K$ GPU blocks to a single tile. The number of blocks equals the number of viewpoints $K$ [S1, unofficial].
3.  **Thread Configuration:** Ensure each block $k$ has exactly $N/K$ threads, matching the number of subsampled pixels for that viewpoint [S1, unofficial].
4.  **Viewpoint Allocation:** Assign each block $k$ to a specific viewpoint $k$ [S1, unofficial].
5.  **Execution:** For each block $k$ from $1$ to $K$, execute the following kernel logic:
    *   Initialize shared memory `Shmem_k` of size $N/K$ [S1, unofficial].
    *   For each thread $i$ from $1$ to $N/K$ in block $k$:
        *   Set `isValid_{k,i}` to `true` [S1, unofficial].
        *   If the pixel coordinate $P_A[k,i]$ is outside the image bounds, set `isValid_{k,i}` to `false` [S1, unofficial].
        *   While there is unfetched data left in the sorted Gaussian array $D_I$ for the assigned viewpoint:
            *   Load data from $D_I$ at index `offset * (N/K) + i` into `Shmem_k[i]` [S1, unofficial].
            *   If `isValid_{k,i}` is `true`, update `V_{k,i}` by calling `alphaBlending(Shmem_k, V_{k,i})` [S1, unofficial].
            *   Synchronize all threads in block $k$ using `synchronize()` [S1, unofficial].
            *   Increment `offset` by $1$ [S1, unofficial].
        *   Assign the final blended value `V_{k,i}` to pixel $P_A[k,i]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel executed by block k (where k ranges from 0 to K-1)
# Each block has N_sub threads, where N_sub = N / K
# Shared memory: Shmem[N_sub]
# Global memory: D_I (sorted Gaussians for viewpoint k), P_A (pixel coordinates)

def thread_efficient_partial_rendering_block(k, N_sub, D_I, P_A):
    # Shared memory allocation (conceptual)
    Shmem = [None] * N_sub
    
    # Thread index within the block
    i = get_thread_index_in_block()
    
    # 1. Initialize validity flag
    is_valid = True
    
    # 2. Check image bounds
    pixel_coord = P_A[k, i]
    if not is_within_image_bounds(pixel_coord):
        is_valid = False
        
    offset = 0
    
    # 3. Process Gaussians
    while data_exists_in_D_I(D_I, offset, N_sub, i):
        # Load Gaussian data into shared memory
        # Index calculation: offset * (N/K) + i
        gaussian_data = get_gaussian_from_sorted_array(D_I, offset * N_sub + i)
        Shmem[i] = gaussian_data
        
        # 4. Alpha Blending if valid
        if is_valid:
            # V_k,i is stored in thread-local or shared memory depending on implementation
            # Assuming V_k,i is accumulated across iterations
            V_k_i = alpha_blending(Shmem, V_k_i)
            
        # 5. Synchronize threads in the block
        synchronize_block()
        
        # 6. Increment offset
        offset += 1
        
    # 7. Write final result
    if is_valid:
        write_to_pixel(pixel_coord, V_k_i)
```

**Edge Cases and Pitfalls:**
*   **Idle Threads:** Naive partial rendering using a binary mask in 3DGS causes idle threads because GPU threads are allocated in blocks. If a tile is masked such that only one pixel is unmasked, all other threads in the block wait, leading to low GPU utilization [S1, unofficial]. This method eliminates that by ensuring every thread in a block has a valid pixel to render [S1, unofficial].
*   **Block Size Alignment:** While 3DGS typically aligns tile size with block size, this method requires decreasing the block size to allow multiple blocks to render a single tile [S1, unofficial].
*   **Precision:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** The method reduces rasterizer overhead compared to naive multi-view training by rendering only a subset of pixels per view, while maintaining the same total number of training pixels as single-view training [S1, unofficial].
*   **Memory Complexity:** Uses shared memory `Shmem_k` of size $N/K$ per block [S1, unofficial].
*   **Performance:** Maximizes GPU utilization by eliminating idle threads that occur in naive binary masking approaches [S1, unofficial].
*   **Strengths:** Effective parallelization for multi-view training; reduces overhead without sacrificing training pixel count [S1, unofficial].
*   **Limitations:** Requires custom kernel implementation to handle multiple blocks per tile and dynamic subsampling indices [S1, unofficial].
*   **When to use:** Multi-view training of 3D Gaussian Splatting where GPU utilization is critical [S1, unofficial].
*   **When not to use:** Single-view training, where standard tiled rendering is sufficient [S1, unofficial].

## Background
The method was introduced in the paper "Efficient multi-view training for 3D Gaussian Splatting" [S1, unofficial]. It addresses the inefficiencies of standard tiled rendering when applied to multi-view scenarios, specifically the problem of thread divergence and idle threads caused by binary masking [S1, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
