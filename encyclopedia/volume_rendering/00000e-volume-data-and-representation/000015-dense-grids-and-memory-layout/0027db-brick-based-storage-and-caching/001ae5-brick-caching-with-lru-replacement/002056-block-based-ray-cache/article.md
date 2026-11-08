# Block-based Ray Cache

Also called: Buffered-block list, Cache Data Structure for Ray Tracing

## Overview
The Block-based Ray Cache is a memory optimization technique for volume rendering that reduces reconstruction overhead by retaining reconstructed voxel blocks in memory for reuse by adjacent rays [S1, unofficial]. It solves the problem of redundant computation in high-resolution volume rendering where data exceeds physical memory, allowing rendering times to drop from days to minutes [S1, unofficial]. It sets itself apart from sibling methods by explicitly maintaining a sorted list of blocks along the ray depth to facilitate immediate lookup, rather than relying solely on generic LRU eviction without spatial locality awareness [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray trace request for a specific voxel coordinate and the current state of the cache [S1, unofficial].
*   **Output:** The voxel value if cached, or a newly reconstructed block added to the cache [S1, unofficial].

**Data Structures:**
The core structure is an auxiliary list called the **buffered-block list** [S1, unofficial].
*   **Node Layout:** Each node in the list represents a reconstructed block [S1, unofficial].
    *   `block_id`: Identifier for the block [G].
    *   `boundaries`: The 3D spatial boundaries (min/max coordinates) of the block [S1, unofficial].
    *   `data`: The voxel values of the reconstructed block [G].
    *   `depth`: A numeric value used to determine the "oldest" block for replacement [G].
*   **Ordering:** The list is maintained in sorted order according to the depth of the blocks along the current ray [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Tracing Initiation:** Begin tracing a projection ray through the volume [S1, unofficial].
2.  **Cache Lookup:** For each voxel access along the ray, search the buffered-block list for a node whose `boundaries` contain the requested voxel coordinate [S1, unofficial].
3.  **Hit:** If found, retrieve the voxel value immediately from the cached block data [S1, unofficial].
4.  **Miss:** If not found, invoke the block reconstruction procedure to generate the block containing the voxel [S1, unofficial].
5.  **Buffering:** After a block is reconstructed, buffer it in cache memory instead of discarding it [S1, unofficial]. Add a new node to the buffered-block list with the block's boundaries and data [S1, unofficial].
6.  **List Maintenance:** Ensure the buffered-block list remains sorted by depth along the ray [S1, unofficial].
7.  **Replacement:** If the cache is full, identify the oldest block in the buffered-block list based on the replacement policy [S1, unofficial]. Remove the oldest node from the list [S1, unofficial]. Substitute the oldest block with the newly reconstructed block [S1, unofficial]. Update the buffered-block array with the new block's boundaries [S1, unofficial].
8.  **Completion:** After the ray tracing is complete, the sequence of blocks reconstructed along that ray remains in the cache for potential use by adjacent rays [S1, unofficial].

**Formulas:**
*   **Cache Size Requirement:** In 3D, for an image resolution of $N \times N$ and a block size of $s \times s \times s$, the total number of blocks to store in the cache is:
    $$ \text{Total Blocks} = 2 \times s \times N \times N $$
    This accounts for two layers of sub-volumes sufficient to cover worst-case access patterns [S1, unofficial].

**Parameters:**
*   $N$: Volume data dimension (e.g., 256, 1024) [S1, unofficial].
*   $s$: Block size dimension (e.g., 16, 32) [S1, unofficial].
*   Typical Cache Sizes:
    *   For $N=256$: 0.5 MB ($s=16$) or 1 MB ($s=32$) [S1, unofficial].
    *   For $N=1024$: 2 MB ($s=16$) or 4 MB ($s=32$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Cache Size Too Small:** If the cache memory size is too small, the hit rate becomes very small, degrading performance [S1, unofficial].
*   **Redundant Reconstruction:** Failing to cache blocks leads to significant performance loss, as voxels reconstructed along one ray are likely used by adjacent rays [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BufferedBlockNode:
    def __init__(self, block_id, boundaries, data):
        self.block_id = block_id
        self.boundaries = boundaries  # Tuple of (min_x, min_y, min_z, max_x, max_y, max_z)
        self.data = data
        self.depth = calculate_depth(boundaries)

def search_cache(buffered_block_list, voxel_coord):
    for node in buffered_block_list:
        if is_voxel_in_block(voxel_coord, node.boundaries):
            return node.data
    return None

def is_voxel_in_block(voxel, boundaries):
    min_x, min_y, min_z, max_x, max_y, max_z = boundaries
    return (min_x <= voxel.x <= max_x and
            min_y <= voxel.y <= max_y and
            min_z <= voxel.z <= max_z)

def replace_oldest(buffered_block_list, new_node):
    if len(buffered_block_list) > 0:
        buffered_block_list.pop(0)
    insert_sorted(buffered_block_list, new_node)

def insert_sorted(block_list, new_node):
    for i, node in enumerate(block_list):
        if new_node.depth < node.depth:
            block_list.insert(i, new_node)
            return
    block_list.append(new_node)

def calculate_depth(boundaries):
    return (boundaries[0] + boundaries[3]) / 2  # Proxy for ray depth [G]

def get_block_id(voxel):
    return (voxel.x // 16, voxel.y // 16, voxel.z // 16)  # Standard grid ID [G]

def reconstruct_block(block_id):
    # Standard multiresolution reconstruction logic [G]
    return [[0.0 for _ in range(16)] for _ in range(16*16)]

def get_block_boundaries(block_id):
    x, y, z = block_id
    return (x*16, y*16, z*16, x*16+16, y*16+16, z*16+16)  # Standard grid bounds [G]

def accumulate(values):
    return sum(values)  # Standard ray accumulation [G]

def render_ray(ray, cache_capacity):
    buffered_block_list = []
    voxel_values = []
    for voxel in ray.voxels:
        cached_data = search_cache(buffered_block_list, voxel)
        if cached_data is not None:
            voxel_values.append(cached_data[voxel.x % 16][voxel.y % 16][voxel.z % 16])
        else:
            block_id = get_block_id(voxel)
            reconstructed_block = reconstruct_block(block_id)
            boundaries = get_block_boundaries(block_id)
            new_node = BufferedBlockNode(block_id, boundaries, reconstructed_block)
            if len(buffered_block_list) >= cache_capacity:
                replace_oldest(buffered_block_list, new_node)
            else:
                insert_sorted(buffered_block_list, new_node)
            voxel_values.append(reconstructed_block[voxel.x % 16][voxel.y % 16][voxel.z % 16])
    return accumulate(voxel_values)
```

## Subtopics

### Block-Wise Reconstruction Cache
A technique to reduce reconstruction overhead by caching reconstructed voxel blocks in memory [S1, unofficial]. It leverages the observation that voxels reconstructed along one ray are likely used by adjacent rays, retaining these blocks to avoid re-reconstruction [S1, unofficial]. Using this cache technique reduces rendering time from a few days to a few minutes [S1, unofficial]. It saves approximately 50% of the reconstruction time compared to point-wise reconstruction [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Lookup is $O(B)$ where $B$ is the number of blocks in the cache, assuming linear search of the buffered-block list [S1, unofficial]. Reconstruction is avoided for cached blocks, significantly reducing total rendering time [S1, unofficial].
*   **Memory Complexity:** The cache size grows at a rate of $2^2$ (quadratic) with respect to resolution $N$, while the original data grows at $2^3$ (cubic) [S1, unofficial]. This makes caching efficient for high-resolution data that exceeds physical memory [S1, unofficial].
*   **Measured Performance:** For $N=256$ and $s=16$, cache size is 0.5 MB; for $N=1024$ and $s=16$, cache size is 2 MB [S1, unofficial]. Rendering time can be reduced from days to minutes [S1, unofficial].
*   **Strengths:** Efficient for high-resolution data; reduces redundant computation; small memory footprint relative to original data [S1, unofficial].
*   **Limitations:** Performance degrades if cache size is too small; requires sorted list maintenance; not given in the sources whether parallel access to the cache is supported [S1, unofficial].
*   **When to use:** Scientific visualization applications with high-resolution volume data that exceeds physical memory [S1, unofficial].
*   **When not to use:** Low-resolution data where reconstruction overhead is negligible; scenarios where cache hit rates are expected to be very low due to poor spatial locality [S1, unofficial].

## Background
The method originates from research on direct rendering of multiresolution decomposed volume data [S1, unofficial]. It was proposed to address issues related to volume rendering of multiresolution data, introducing a novel abstract mathematical definition of the volume rendering equation [S1, unofficial]. The technique builds on the observation that block-wise reconstruction is more efficient than point-wise reconstruction [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
