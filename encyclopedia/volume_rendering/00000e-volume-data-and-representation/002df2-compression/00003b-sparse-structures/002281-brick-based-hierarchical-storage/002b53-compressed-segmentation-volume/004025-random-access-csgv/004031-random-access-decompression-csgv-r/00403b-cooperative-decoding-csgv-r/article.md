# Cooperative Decoding (CSGV-R)
Also called: Brick-wise Decompression Stage, brick cache (sm)

## Overview
Cooperative decoding is a GPU shader stage for Compressed Segmentation Volume Random Access (CSGV-R) where a single wavefront cooperatively decompresses one brick, eliminating the thread divergence inherent in serial CSGV decoding [S1, unofficial]. It enables true random access rendering without the memory overhead and latency of a brick cache by leveraging shared memory for the wavelet tree [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Compressed brick data (wavelet tree) in global memory, requested Level of Detail (LOD), and voxel coordinates within the brick.
*   Output: Decompressed volume label for the target voxel.

**Data Structures:**
*   **Shared Memory Cache:** A buffer sized to hold the wavelet tree of the current brick. This is referred to as "brick cache (sm)" [S1, unofficial].
*   **Wavelet Tree:** The compressed representation of the brick's labels, stored in global memory and copied to shared memory [S1, unofficial].

**Steps:**
1.  **Identify Brick:** The thread block (wavefront) identifies the brick containing the requested voxel.
2.  **Load to Shared Memory:** The entire wavelet tree for the identified brick is copied from global memory to shared memory [S1, unofficial]. This step is critical for performance [S1, unofficial].
3.  **Cooperative Decompression:** Threads within the wavefront cooperate to decompress the brick.
    *   Threads iterate over output voxels in the requested LOD.
    *   Each thread reads its specific volume label using the random access mechanism provided by CSGV-R [S1, unofficial].
    *   The decomposition logic utilizes the wavelet tree stored in shared memory to resolve the label for the specific voxel coordinate [S1, unofficial].
4.  **Return Label:** The decompressed label is returned to the rendering pipeline [S1, unofficial].

**Parameters:**
*   **LOD:** The level of detail requested by the renderer. Coarser LODs are faster to access [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Shared Memory Size:** The shared memory must be large enough to hold the wavelet tree for the largest brick size supported. If the tree exceeds shared memory limits, the optimization fails [G].
*   **Wavefront Size:** The method assumes a wavefront size that matches or exceeds the number of voxels being cooperatively processed or the number of threads required for the decomposition logic [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume this runs in a GPU shader kernel for a wavefront processing one brick
def cooperative_decode_brick(brick_id, target_voxel_coords, lod):
    # 1. Determine the brick's wavelet tree data location in global memory
    global_tree_ptr = get_brick_global_memory_address(brick_id)
    
    # 2. Copy wavelet tree to shared memory (brick cache (sm))
    #    This is done cooperatively by the wavefront
    shared_tree = copy_to_shared_memory(global_tree_ptr, get_brick_tree_size(brick_id))
    
    # 3. Cooperative decomposition
    #    Threads in the wavefront work together to resolve the label
    #    for the target voxel at the specified LOD
    
    # Get the label using CSGV-R random access on the shared tree
    label = csgv_r_random_access_decode(shared_tree, target_voxel_coords, lod)
    
    return label
```

## Subtopics

### Brick Caching (CSGV)
CSGV supports caching full bricks decompressed by a separate shader up to a requested LOD with a one-frame latency [S1, unofficial]. Cache regions are assigned as in shading atlas streaming [S1, unofficial]. When a brick is first accessed and not present in the cache, a random sample from the brick palette or the most frequent label (first palette entry) is used as a placeholder [S1, unofficial]. This results in visible popping artifacts [S1, unofficial]. The CSGV cache barely fits all bricks for ambient occlusion configurations, and deeper traversal in path tracing will not allow decoding all visible voxels [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** CSGV-R with cooperative decoding has no extra memory overhead during rendering beyond the VRAM required for the compressed volume itself [S1, unofficial]. In contrast, CSGV requires extra memory for the cache of visible bricks [S1, unofficial].
*   **Time:** Cooperative decoding avoids thread divergence caused by serial decoding (one thread per brick accessing different global memory regions) [S1, unofficial].

**Measured Performance:**
Performance is measured in frame times (ms) for local shading [S1, unofficial]:

| Dataset | CSGV rANS (brick cache) | CSGV-R (brick cache (sm)) |
| :--- | :--- | :--- |
| CELLS | 11.8 ms | 8.1 ms |
| FIBER | 6.0 ms | 4.5 ms |
| H01 | 4.1 ms | Not given in the sources. |
| AZBA | 10.3 ms | 9.3 ms |

For the FIBER dataset, cooperative decoding of bricks achieves the fastest rendering [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Eliminates thread divergence, removes brick cache memory overhead, avoids popping artifacts associated with CSGV caching, and provides faster rendering for certain datasets (e.g., FIBER) [S1, unofficial].
*   **Limitations:** Requires sufficient shared memory for the wavelet tree. Performance gains are dataset-dependent [S1, unofficial].

**When to Use:**
Use when rendering CSGV-R volumes on GPUs with sufficient shared memory and when avoiding thread divergence and cache overhead is critical [S1, unofficial].

## Background
Cooperative decoding was introduced as an alternative brick-wise decompression shader stage for CSGV-R in the paper "Random Access Segmentation Volume Compression for Interactive Volume Rendering" [S1, unofficial]. It addresses the limitations of the original CSGV approach, which suffered from thread divergence and cache latency [S1, unofficial].

## Sources
[S1] Random Access Segmentation Volume Compression for Interactive Volume Rendering, https://doi.org/10.1111/cgf.70116, fetched 2026-10-28 (unofficial)
