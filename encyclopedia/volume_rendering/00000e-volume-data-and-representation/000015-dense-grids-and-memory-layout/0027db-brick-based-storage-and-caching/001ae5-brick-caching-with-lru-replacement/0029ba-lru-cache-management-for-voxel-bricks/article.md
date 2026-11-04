# LRU Cache Management for Voxel Bricks

Also called: Brick Cache Management, Gestion du cache de briques, LRU Brick Cache, Last Recently Used cache strategy, Stratégie LRU

## Overview
LRU Cache Management for Voxel Bricks is a memory management strategy that prioritizes reusing voxel brick data that has not been accessed for the longest time, thereby optimizing memory reuse while retaining useful data in the cache [S1, S2, unofficial]. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized [S1, unofficial]. Unlike sibling methods that may rely on hardware-specific features or different replacement policies, this approach explicitly manages the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** The current visibility state of bricks (determined by a visibility determination phase), the current frame number (global timestamp), and the brick data to be loaded.
*   **Outputs:** An updated GPU brick cache containing the most recently used bricks, and an updated LRU buffer reflecting the access order.

**Data Structures**
The implementation requires maintaining parallel structures on the CPU and GPU [S1, S2, unofficial].

1.  **CPU-side Structures [S1, unofficial]:**
    *   **Parallel N3-tree Indirection:** A copy of the hierarchical storage structure (N3-tree) in central memory to allow CPU-side manipulation and notification of node destruction.
    *   **Usage Flag:** A flag maintained for each loaded brick at each Level of Detail (LOD). It stores the last time of use to determine priority in the LRU cache.
    *   **Ordered List:** A list sorted by last usage time, used to identify the next free cache slot or the victim slot for eviction.
    *   **Owner Node Structure:** A structure similar to the brick indirection storage, allowing direct access to owner node data to notify them of data destruction.

2.  **GPU-side Structures [S2, unofficial]:**
    *   **Brick Cache:** A single-channel 3D texture in `r8unorm` format. Each entry corresponds to a single brick. The cache size $C_{size}$ is chosen by the user at initialization and rounded up to be a multiple of the brick size $B_{size}$.
    *   **LRU Buffer:** A GPU storage buffer containing an array of linear indices into the brick cache, sorted by most recent access time.
    *   **Usage Buffer:** A 3D texture in `r32uint` format with one voxel per brick-cache entry. Its size $U_{size}$ is defined as:
        $$
        U_{size} = \left( \frac{C_{size_x}}{B_{size_x}}, \frac{C_{size_y}}{B_{size_y}}, \frac{C_{size_z}}{B_{size_z}} \right)
        $$
        This buffer stores the timestamp (frame number) when a brick-cache entry was last accessed [S2, unofficial].

**Numbered Steps**
1.  **Visibility Determination:** During the visibility determination phase, the system identifies which bricks are needed for the current frame [S1, unofficial].
2.  **Update Usage Flags (CPU) / Usage Buffer (GPU):**
    *   *CPU Approach [S1, unofficial]:* The usage flag for each loaded brick is updated with the current time/frame number.
    *   *GPU Approach [S2, unofficial]:* During rendering, whenever a brick-cache entry is accessed, the current global timestamp (frame number) is written into the corresponding voxel in the Usage Buffer.
3.  **Check Cache Status:** When a new brick or brick resolution is needed:
    *   If the cache is not full, transfer the brick to the next free slot [S1, unofficial].
    *   If the cache is full, identify the slot containing the brick that has not been used for the longest time [S1, unofficial].
4.  **Eviction and Update:**
    *   If a slot is reused because it contains an old brick, and that brick is still referenced by the storage structure, update the hierarchical structure to invalidate the corresponding LOD level in the leaf containing the brick [S1, unofficial].
    *   *GPU LRU Update [S2, unofficial]:* After each frame, a series of WebGPU compute passes update the LRU Buffer. The Usage Buffer is processed to produce a mask of accessed entries, which are then sorted to update the LRU Buffer array [S2, unofficial].
5.  **Data Transfer:** Transfer the new brick data to the GPU using the graphics API (e.g., `glTexSubImage` in OpenGL) [S1, unofficial]. Asynchronous transfers can be managed using Pixel Buffer Objects (PBOs) [S1, unofficial].

**Pseudocode**
The following pseudocode illustrates the CPU-side logic for cache management, as described in [S1, unofficial]. Note that GPU-side management (as in [S2, unofficial]) uses compute shaders to update the LRU buffer based on the usage buffer.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def manage_lru_cache(brick_needed, cache_slots, usage_flags, hierarchical_structure):
    """
    brick_needed: The brick data and metadata (LOD, position) to be loaded.
    cache_slots: List of available cache slots.
    usage_flags: Dictionary mapping brick_id -> last_used_time.
    hierarchical_structure: The N3-tree copy for invalidation.
    """
    # Step 1: Check if cache has free space
    if len(cache_slots) > 0:
        # Case: Cache is not full
        slot = cache_slots.pop(0) # Take next free slot
        transfer_brick_to_gpu(slot, brick_needed)
        usage_flags[brick_needed.id] = current_time()
        return slot
    else:
        # Case: Cache is full
        # Find the brick with the oldest usage time
        victim_id = min(usage_flags, key=usage_flags.get)
        victim_slot = get_slot_for_brick(victim_id)
        
        # Check if victim brick is still referenced
        if is_brick_referenced(victim_id, hierarchical_structure):
            # Invalidate the corresponding LOD level in the leaf
            hierarchical_structure.invalidate_lod(victim_id)
        
        # Reuse the slot
        cache_slots.append(victim_slot) # Or manage slot list appropriately
        transfer_brick_to_gpu(victim_slot, brick_needed)
        
        # Update usage flags
        del usage_flags[victim_id]
        usage_flags[brick_needed.id] = current_time()
        
        return victim_slot
```

**Parameters**
*   **Cache Size ($C_{size}$):** Chosen by the user at initialization. It must be rounded up to be a multiple of the brick size $B_{size}$ [S2, unofficial].
*   **Usage Flag:** Maintained on the CPU (or in a GPU Usage Buffer) for each loaded brick at each LOD level, storing the last time of use [S1, S2, unofficial].

**Edge Cases and Pitfalls**
*   **GPU Random Writes:** Standard shader stream processing models do not allow random memory writes, making pure GPU-side LRU management difficult without specific APIs like CUDA or compute shaders with atomic operations [S1, unofficial]. The solution is to manage the cache on the CPU or use compute passes to update buffers [S1, S2, unofficial].
*   **Invalidation:** If a reused brick is still referenced by the storage structure, the hierarchical structure must be updated to invalidate the corresponding LOD level; failure to do so leads to rendering artifacts or stale data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The LRU update step requires sorting or scanning the usage buffer. In the CPU approach, finding the minimum usage time is $O(N)$ where $N$ is the number of cached bricks. In the GPU approach, compute passes are required each frame to update the LRU buffer [S2, unofficial].
*   **Memory Complexity:** Requires maintaining parallel structures on CPU (N3-tree copy, ordered list, owner node structure) and GPU (Brick Cache, LRU Buffer, Usage Buffer) [S1, S2, unofficial].
*   **Strengths:** Optimizes memory reuse by prioritizing data that is likely to be reused soon, effectively managing limited GPU memory transfer speeds and capacity [S1, unofficial].
*   **Limitations:** The CPU-side management adds overhead for synchronization and flag updates. The GPU-side approach requires additional compute passes and buffer management [S1, S2, unofficial].
*   **When to Use:** When GPU memory is constrained and data access patterns exhibit temporal locality, making LRU an effective replacement policy.
*   **When Not to Use:** If the access pattern is random or if the overhead of maintaining LRU state outweighs the benefits of cache hits.

## Background
The LRU strategy for voxel bricks was introduced in the context of interactive exploration of procedural volumetric data [S1, unofficial]. It addresses the bottleneck of transferring data from slow host memory to fast GPU memory [S1, unofficial]. The approach was further implemented in web-based volume rendering systems for large biomedicinal datasets, utilizing WebGPU compute passes for LRU buffer updates [S2, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
