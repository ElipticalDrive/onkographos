# LRU Cache Management for Voxel Bricks

Also called: LRU Brick Cache, LRU Cache Management, Gestion du cache de briques, Stratégie LRU, Last Recently Used cache strategy

## Overview
LRU Cache Management for Voxel Bricks is a memory management strategy for out-of-core volume rendering that prioritizes the retention of recently accessed voxel bricks to optimize memory reuse [S1, S3, unofficial]. It solves the problem of limited GPU memory capacity and slow data transfer rates by ensuring that the most frequently or recently needed brick data remains in the fast working set [S1, unofficial]. Unlike sibling methods that may rely on complex prediction models or GPU-steered updates, this approach typically manages the cache state on the CPU to bypass shader memory write limitations, or uses specific GPU buffers to track access timestamps [S1, S2, unofficial].

## How it works
The method manages a fixed-size cache of voxel bricks, where each entry corresponds to a specific brick at a specific Level of Detail (LOD). The core logic relies on tracking the "last used" time for each brick to determine eviction priority.

### Inputs and Outputs
*   **Input:** A request for a specific brick (identified by coordinates and LOD level); the current state of the cache (full or empty).
*   **Output:** A valid brick data block in the cache ready for rendering, or an eviction of the least recently used brick to make space.

### Data Structures
The implementation requires specific data structures to track brick locations and access times. Two primary variants exist in the literature:

**Variant A: CPU-Managed (Sarton et al.)**
1.  **Parallel N3-Tree Copy:** A copy of the hierarchical indirection storage (N3-tree) resides in CPU memory to allow direct access to owner node data [S1, unofficial].
2.  **Usage Flag:** A flag maintained on the CPU for each loaded brick at each LOD level, storing the last time of use [S1, unofficial].
3.  **Ordered List:** A list sorted by last usage time, maintained in CPU memory, used to quickly identify the next free cache slot or the brick to evict [S1, unofficial].
4.  **Owner Node Structure:** A structure similar to the brick indirection storage, allowing direct access to the owner node to notify it of data destruction [S1, unofficial].

**Variant B: GPU-Managed with Buffers (Skalierbares Webbasiertes)**
1.  **Brick Cache:** A single-channel 3D texture in `r8unorm` format. Each entry corresponds to a single brick [S2, unofficial].
2.  **LRU Buffer:** A GPU storage buffer containing an array of linear indices into the brick cache, sorted by most recent access time (most recent to least recent) [S2, unofficial].
3.  **Usage Buffer:** A 3D texture in `r32uint` format with one voxel per brick-cache entry. It stores the last access timestamp for each entry in the brick cache [S2, unofficial].
    *   Size $U_{size}$ is defined as:
        $$ U_{size} = \left( \frac{C_{size_x}}{B_{size_x}}, \frac{C_{size_y}}{B_{size_y}}, \frac{C_{size_z}}{B_{size_z}} \right) $$
        where $C_{size}$ is the cache size and $B_{size}$ is the brick size [S2, unofficial].

### Parameters
*   **Cache Size ($C_{size}$):** The maximum number of bricks in the working set. In GPU implementations, this is rounded up to be a multiple of the brick size $B_{size}$ [S2, unofficial].
*   **Brick Size ($B_{size}$):** The dimensions of a single brick in voxels [S2, unofficial].
*   **Timestamp:** A global counter (e.g., frame number) used to record access times [S2, unofficial].

### Steps
The algorithm proceeds as follows when a brick is requested:

1.  **Check Cache Status:** Determine if the requested brick is already in the cache.
    *   If **Yes**: Update the usage timestamp for this brick.
        *   *CPU Variant:* Update the usage flag on the CPU and reorder the ordered list [S1, unofficial].
        *   *GPU Variant:* Write the current timestamp to the corresponding location in the Usage Buffer [S2, unofficial].
    *   If **No**: Proceed to load the brick.

2.  **Load Brick:**
    *   **If Cache is Not Full:** Transfer the new brick to the next free slot in the cache [S1, unofficial].
        *   *CPU Variant:* Use the ordered list to find the next free slot [S1, unofficial].
        *   *GPU Variant:* Assign a free index in the LRU Buffer [S2, unofficial].
    *   **If Cache is Full:** Identify the slot containing the brick that has not been used for the longest time (the LRU brick) and reuse that slot [S1, unofficial].
        *   *CPU Variant:* The last element in the ordered list represents the LRU brick [S1, unofficial].
        *   *GPU Variant:* The last element in the LRU Buffer is the LRU index [S2, unofficial].

3.  **Update Hierarchy (CPU Variant Only):** If the reused brick was still referenced by the storage structure (e.g., the N3-tree), update the hierarchical structure to invalidate the corresponding LOD level in the leaf containing the brick [S1, unofficial].

4.  **Update LRU Order (GPU Variant Only):** After rendering, update the LRU Buffer based on the Usage Buffer.
    *   Generate a mask $M$ from the Usage Buffer marking indices accessed at the current timestamp $t$.
    *   Identify $L_u^+$, the indices in the old LRU buffer ($L_o$) marked in $M$.
    *   Identify $L_u^-$, the indices in $L_o$ not marked in $M$.
    *   Construct the Updated LRU Buffer by concatenating $L_u^+$ followed by $L_u^-$ [S2, unofficial]. This preserves the relative order of accessed bricks and unaccessed bricks, moving accessed bricks to the front [S2, unofficial].

### Pseudocode
The following pseudocode combines the CPU-managed logic (primary implementation in [S1, unofficial]) with the GPU buffer update logic (from [S2, unofficial]) for completeness.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global State
cache = {}          # Map: BrickID -> Data
lru_list = []       # List of BrickIDs, sorted most-recent to least-recent
usage_flags = {}    # Map: BrickID -> Timestamp
cache_capacity = 100 # Example capacity

def request_brick(brick_id, current_timestamp):
    """
    Requests a brick. Returns the brick data.
    """
    if brick_id in cache:
        # Brick is in cache, update usage
        update_usage(brick_id, current_timestamp)
        return cache[brick_id]
    
    # Brick not in cache, need to load
    if len(cache) < cache_capacity:
        # Cache not full, add to next free slot
        load_brick(brick_id, current_timestamp)
    else:
        # Cache full, evict LRU
        lru_brick_id = lru_list[-1] # Last element is least recent
        evict_brick(lru_brick_id)
        load_brick(brick_id, current_timestamp)
    
    return cache[brick_id]

def update_usage(brick_id, timestamp):
    """
    Updates the usage timestamp and reorders the LRU list.
    """
    usage_flags[brick_id] = timestamp
    # Remove from current position and append to front
    if brick_id in lru_list:
        lru_list.remove(brick_id)
    lru_list.insert(0, brick_id)

def load_brick(brick_id, timestamp):
    """
    Loads brick from storage and adds to cache.
    """
    # [G] Fetch brick data from disk or parent memory hierarchy
    data = fetch_brick_from_storage(brick_id)
    cache[brick_id] = data
    usage_flags[brick_id] = timestamp
    lru_list.insert(0, brick_id)
    
    # [S1, unofficial] If using N3-tree, invalidate LOD in owner node
    invalidate_lod_in_hierarchy(brick_id)

def evict_brick(brick_id):
    """
    Removes brick from cache and updates structures.
    """
    del cache[brick_id]
    lru_list.remove(brick_id)
    del usage_flags[brick_id]
    # [S1, unofficial] Notify owner node of destruction
    notify_owner_node_destruction(brick_id)

# GPU-Specific Update Logic (from [S2, unofficial])
def update_gpu_lru_buffer(lru_buffer, usage_buffer, timestamp):
    """
    Updates the GPU LRU buffer based on usage buffer.
    lru_buffer: List of indices sorted by recency.
    usage_buffer: Array of timestamps for each cache entry.
    timestamp: Current frame timestamp.
    """
    # 1. Generate mask M from usage buffer for timestamp t
    # Indices where usage_buffer[i] == timestamp
    accessed_indices = [i for i, t in enumerate(usage_buffer) if t == timestamp]
    
    # 2. Lu+ = indices in Old LRU (Lo) marked in M
    # Note: lru_buffer contains indices into the cache. 
    # We need to find which of those indices were accessed.
    accessed_in_lru = [idx for idx in lru_buffer if idx in accessed_indices]
    
    # 3. Lu- = indices in Lo not marked in M
    unaccessed_in_lru = [idx for idx in lru_buffer if idx not in accessed_indices]
    
    # 4. Updated LRU = Lu+ concatenated with Lu-
    new_lru_buffer = accessed_in_lru + unaccessed_in_lru
    
    return new_lru_buffer
```

### Edge Cases and Pitfalls
*   **GPU Random Writes:** Standard graphics shaders do not allow random memory writes, making pure GPU-side LRU management difficult without compute shaders or specific extensions [S1, unofficial]. The CPU-managed approach avoids this by handling the logic on the CPU and using APIs like `glTexSubImage` for data transfer [S1, unofficial].
*   **Visibility Stalls:** Caching alone may not deliver smooth frame rates if visibility changes cause bursts of disk activity. Small changes in visibility can lead to high variance in frame rates [S3, unofficial].
*   **Precision:** Timestamps must be unique enough to distinguish access order. Using frame numbers is common [S2, unofficial].

## Subtopics

### [LRU Cache Management for Octree Nodes](11658-lru-cache-management-for-octree-nodes.md)
LRU Cache Management for Octree Nodes is a demand-based page replacement algorithm used to manage the caching of octree nodes in out-of-core volume rendering systems. It solves the problem of efficiently managing limited in-core memory by prioritizing the retention of recently accessed nodes, thereby optimizing memory reuse. Unlike specialized predictors, it is a generic strategy with minimal application-specific logic, serving as a baseline for evaluating more complex prefetching algorithms.

### Editable Voxel Lifecycle with Regenerable Cache
A lifecycle management strategy where unedited bricks are treated as a regenerable cache that can be evicted, while edited bricks are persisted as full-resolution snapshots. This approach distinguishes itself by allowing the cache to "regenerate" data for unmodified regions, reducing storage requirements for large, mostly static datasets. The edit wire format is defined separately in the network architecture context [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Cache lookup is $O(1)$ if using a hash map for brick IDs. Updating the LRU order is $O(N)$ where $N$ is the cache size, due to list reordering or buffer updates [S1, S2, unofficial].
*   **Memory Complexity:** Requires storing the cache data plus auxiliary structures (usage flags, LRU lists). The GPU variant requires additional buffers (LRU Buffer, Usage Buffer) proportional to the cache size [S2, unofficial].
*   **Performance:** Effective when there is locality of reference, i.e., changes in visibility from frame to frame are small [S3, unofficial]. Performance degrades if visibility changes are large or random, leading to thrashing [S3, unofficial].
*   **Strengths:** Simple to implement; effective for static or slowly changing scenes; minimizes redundant data transfers [S1, unofficial].
*   **Limitations:** Does not predict future access; susceptible to thrashing under high visibility variance; CPU management adds overhead for data transfer [S1, S3, unofficial].

## Background
The LRU strategy for geometry caching was introduced as a simple and effective approach for out-of-core visualization of large datasets [S3, unofficial]. It is based on the principle that keeping the least-recently used pieces of geometry in main memory is effective if the visible dataset fits in the cache and exhibits locality of reference [S3, unofficial]. The specific application to voxel bricks in N3-trees was detailed by Sarton et al. [S1, S2, unofficial]. The concept has roots in operating system page replacement algorithms [S3, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
[S4] Choose voxel data structure & LOD scheme #9, https://github.com/theHimanshuShekhar/bhayanakvoxel/issues/9, fetched 2026-10-28 (unofficial)
