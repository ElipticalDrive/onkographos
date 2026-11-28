# Block-based Ray Cache

Also called: Buffered-block list, Cache Data Structure for Ray Tracing

## Overview
The Block-based Ray Cache is a memory management technique for volume rendering that reduces reconstruction overhead by caching reconstructed voxel blocks in memory for reuse by adjacent rays [S1, unofficial]. It solves the problem of redundant computation in high-resolution volume rendering where data exceeds physical memory, by maintaining a sequence of blocks along completed rays and retrieving them for subsequent ray traces [S1, unofficial]. It distinguishes itself from standard spatial traversal by decoupling the processing order of blocks from the ray order within a frame, relying on a sorted auxiliary list structure to ensure cache hits for spatially coherent rays [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset (potentially multiresolution), a ray traversal request for a specific voxel or ray segment, and the current state of the cache.
*   **Output:** The voxel value (if cached) or a reconstructed block added to the cache.

**Data Structures:**
The core data structure is the **buffered-block list**.
*   **Type:** A sorted linked list or array of nodes.
*   **Node Layout:** Each node contains the spatial boundaries (min/max coordinates) of a reconstructed block currently residing in the cache [S1, unofficial].
*   **Ordering:** The list is maintained in sorted order along the ray depth (front-to-back) [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Tracing Initiation:** Begin tracing a projection ray through the volume.
2.  **Cache Lookup:** For each voxel access along the ray:
    a.  Search the buffered-block list for a node whose boundaries contain the requested voxel [S1, unofficial].
    b.  **Hit:** If the voxel is within a cached block, retrieve the value immediately from the cache memory [S1, unofficial].
    c.  **Miss:** If the voxel is not found in any cached block, invoke the reconstruction procedure to generate the block containing the voxel [S1, unofficial].
3.  **Buffering:** After a block is reconstructed, buffer it in cache memory instead of discarding it [S1, unofficial]. Add a new node to the buffered-block list with the block's boundaries [S1, unofficial].
4.  **Ray Completion:** When the tracing of one projection ray is completed, the sequence of blocks reconstructed along that ray remains temporarily maintained in the cache memory for potential use by adjacent rays [S1, unofficial].
5.  **Replacement Policy:** When the cache memory is full:
    a.  Identify the oldest block in the cache (based on insertion time or list order) [S1, unofficial].
    b.  Substitute the oldest block with the new block [S1, unofficial].
    c.  Update the buffered-block array/list to reflect the new block's boundaries and remove the old block's entry [S1, unofficial].

**Formulas:**
*   **Cache Size Requirement:** In 3D, for an image resolution of $N \times N$ and a block size of $s \times s \times s$, the total number of blocks to store is sufficient to cover two layers of sub-volumes in the worst case. The total cache size in terms of blocks is:
    $$ \text{Blocks} = 2 \times s \times N \times N $$
    This assumes two layers of sub-volumes are sufficient to avoid redundant block reconstructions [S1, unofficial].

**Parameters:**
*   **Block Size ($s$):** Typical values used in experiments are $s=16$ and $s=32$ [S1, unofficial].
*   **Resolution ($N$):** Tested resolutions include $N=256$ and $N=1024$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Cache Size Too Small:** If the cache memory size is too small, the ray tracing hit rate will be very small, causing performance to degrade to that of uncached rendering [S1, unofficial].
*   **Block Level Selection:** The multiresolution decomposition level should not be too low (small blocks), as this increases overhead without ensuring blocks contain completely zero voxels; 1 to 3 levels are suitable choices [S1, unofficial].
*   **Boundary Conditions:** The method handles the "extended block boundary problem" associated with block-wise reconstruction, ensuring random access to compressed data is managed correctly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BufferedBlockNode:
    def __init__(self, min_coords, max_coords, block_data):
        self.min_coords = min_coords
        self.max_coords = max_coords
        self.block_data = block_data
        self.timestamp = current_global_timestamp()

class BlockBasedRayCache:
    def __init__(self, max_cache_size):
        self.max_cache_size = max_cache_size
        self.cache_list = []  # Sorted list of BufferedBlockNode
        self.timestamp_counter = 0

    def get_voxel(self, voxel_coords, reconstruct_block_func):
        """
        Retrieve a voxel value. If not in cache, reconstruct and cache it.
        """
        # 1. Search the cache using the buffered-block list
        for node in self.cache_list:
            if self.is_voxel_in_block(voxel_coords, node.min_coords, node.max_coords):
                # 2. If the voxel is within a cached block, retrieve it immediately
                return self.get_voxel_from_block(voxel_coords, node.block_data)
        
        # 3. If not, invoke reconstruction and buffer the new block
        block_data = reconstruct_block_func(voxel_coords)
        block_min, block_max = self.get_block_boundaries(voxel_coords, block_data)
        new_node = BufferedBlockNode(block_min, block_max, block_data)
        
        # Add to cache
        self.add_to_cache(new_node)
        
        return self.get_voxel_from_block(voxel_coords, new_node.block_data)

    def add_to_cache(self, node):
        # 4. Replacement policy: When cache is full, substitute the oldest block
        if len(self.cache_list) >= self.max_cache_size:
            # Find the oldest block (simple LRU based on timestamp)
            oldest_node = min(self.cache_list, key=lambda n: n.timestamp)
            self.cache_list.remove(oldest_node)
        
        node.timestamp = self.timestamp_counter
        self.timestamp_counter += 1
        self.cache_list.append(node)
        # Maintain sorted order along the ray depth if required by traversal logic
        # Note: The source specifies the list is sorted by depth along the ray.
        # In a general cache, strict spatial sorting might be complex; 
        # however, for the specific ray-tracing context described, 
        # we assume the list is managed to facilitate depth-based access.
        self.cache_list.sort(key=lambda n: n.min_coords[2]) # Assuming Z is depth

    def is_voxel_in_block(self, voxel, min_c, max_c):
        return (min_c[0] <= voxel[0] <= max_c[0] and
                min_c[1] <= voxel[1] <= max_c[1] and
                min_c[2] <= voxel[2] <= max_c[2])

    def get_voxel_from_block(self, voxel, block_data):
        # Extract voxel from block data based on local coordinates
        local_coords = self.get_local_coords(voxel, block_data)
        return block_data[local_coords]

    def get_block_boundaries(self, voxel, block_data):
        # Determine the min/max coordinates of the block containing the voxel
        # Implementation depends on the specific block structure
        pass

    def get_local_coords(self, voxel, block_data):
        # Convert global voxel coordinates to local block indices
        pass
```

## Subtopics

### [Trashless Traversal with Block Lists](13631-trashless-traversal-with-block-lists.md)
Trashless Traversal with Block Lists is a cache-oblivious traversal strategy for volume rendering that eliminates cache trashing by ensuring that any data block evicted from the cache is never accessed again during the rendering of a frame. It achieves this by subdividing the volume into blocks, sorting them front-to-back, and organizing them into disjoint block lists such that no single ray intersects more than one block within the same list. This approach distinguishes itself from standard spatial traversal by decoupling the processing order of blocks from the ray order within a frame, allowing for parallel processing of blocks within a list while maintaining strict cache locality.

### Block-Wise Reconstruction Cache
Block-Wise Reconstruction Cache is a technique to reduce reconstruction overhead by caching reconstructed voxel blocks in memory. Since voxels reconstructed along one ray are likely used by adjacent rays, the cache retains these blocks to avoid re-reconstruction [S1, unofficial]. Using this cache technique reduces rendering time from a few days to a few minutes [S1, unofficial]. It saves approximately 50% of the reconstruction time compared to point-wise reconstruction techniques [S1, unofficial].

### Block-based Volume Rendering with Distributed Caching
This variant extends caching to distributed architectures. Optimal block size is between 0.5 and 1.0 times the size of the L2 cache [S2]. For the Power Challenge, with a 4 MB L2 cache, blocks are sized between 2 MB and 4 MB [S2]. This method was implemented on a Silicon Graphics Power Challenge Array, a distributed array of up to eight Power Challenge shared-memory multiprocessors, each with up to 18 MIPS R8000 processors, 16 KB L1 cache, and 4 MB L2 cache, connected by a HIPPI network [S2]. It successfully rendered a 7.1 GB dataset [S2]. This work builds upon previous results by the same authors which optimized memory hierarchy on a single R8000 processor and within a single shared-memory Power Challenge multiprocessor [S2].

## Operating characteristics
*   **Time Complexity:** The cache lookup is $O(K)$ where $K$ is the number of blocks in the cache, assuming a linear search of the buffered-block list. With sorting, binary search could reduce this to $O(\log K)$, though the source describes a simple replacement policy and list update [S1, unofficial]. The primary benefit is the reduction in reconstruction time, which is the dominant cost in multiresolution rendering.
*   **Memory Complexity:** The cache size grows at a rate of $O(N^2)$ (specifically $2 \cdot s \cdot N^2$ blocks) while the original data grows at $O(N^3)$ [S1, unofficial]. This makes caching efficient for high-resolution data that exceeds physical memory.
*   **Measured Performance:** For $N=256$, cache size is 0.5 MB ($s=16$) or 1 MB ($s=32$). For $N=1024$, cache size is 2 MB ($s=16$) or 4 MB ($s=32$) [S1, unofficial]. Rendering time was reduced from a few days to a few minutes using the cache [S1, unofficial].
*   **Strengths:** Significantly reduces reconstruction overhead; handles data sets larger than physical memory; effective for high-resolution data.
*   **Limitations:** Performance degrades if cache size is too small; requires careful selection of multiresolution levels; simple LRU replacement may not be optimal for all access patterns.
*   **When to use:** High-resolution volume rendering where reconstruction is expensive and data exceeds available RAM.
*   **When not to use:** Low-resolution data where reconstruction is cheap, or when cache size is severely constrained.

## Background
The Block-based Ray Cache was introduced in the context of direct rendering of multiresolution decomposed volume data [S1, unofficial]. It addresses the issue of redundant block reconstructions by maintaining relevant reconstructed voxels in a cache data structure [S1, unofficial]. The technique was developed to handle scientific visualization applications where high-resolution volume data sets are often far beyond the available physical memory [S1, unofficial]. The distributed variant was developed to exploit deep parallel memory hierarchies on architectures like the Silicon Graphics Power Challenge Array [S2].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Exploiting deep parallel memory hierarchies for ray casting volume rendering, https://doi.org/10.1145/266638.266645, fetched 2026-10-26
