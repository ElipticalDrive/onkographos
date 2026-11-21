# Brick Occupancy Recomputation
Also called: Brick Occupancy Compute
## Overview
Brick Occupancy Recomputation is a GPU compute shader routine that regenerates the occupancy status of volumetric bricks after dynamic voxel changes, ensuring the hierarchical traversal structure remains consistent with the underlying voxel data [S1, unofficial]. It differs from static generation methods by being designed for frequent, localized updates in interactive simulations, relying on shared memory reduction to avoid atomic operations and global memory contention [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A texture containing voxel data (1-bit solid/air flags) organized into fixed-size bricks [S1, unofficial].
*   **Output:** An occupancy map (brickmap) where each entry corresponds to a brick, storing the count of non-air voxels within that brick [S1, unofficial].

**Data Structures:**
*   **Voxel Texture:** 1-bit per voxel representation of solid vs. air [S1, unofficial].
*   **Occupancy Map:** Array of unsigned integers, one per brick [S1, unofficial].
*   **Shared Memory:** Array of 32 unsigned integers, one per thread in the block [S1, unofficial].

**Parameters:**
*   **Thread Block Size:** Exactly 32 threads [S1, unofficial].
*   **Voxels per Thread:** 16 voxels [S1, unofficial].
*   **Brick Size:** Implicitly $32 \times 16 = 512$ voxels per brick [S1, unofficial].

**Steps:**
1.  **Dispatch:** Launch a compute shader where each thread block is assigned to process exactly one brick [S1, unofficial].
2.  **Load:** Each of the 32 threads reads 16 voxels from its assigned segment of the brick [S1, unofficial].
3.  **Count:** Each thread counts the number of voxels in its segment that are not air (i.e., are solid) [S1, unofficial].
4.  **Store:** Each thread writes its count to a unique index in a shared memory array of unsigned integers [S1, unofficial].
5.  **Sync:** Execute a barrier synchronization to ensure all threads have written their partial counts to shared memory [S1, unofficial].
6.  **Reduce:** The first thread (thread ID 0) sums the values from the shared memory array [S1, unofficial].
7.  **Write:** Thread 0 writes the total sum to the corresponding entry in the brickmap [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute shader entry point: one block per brick, 32 threads per block
def brick_occupancy_recompute(brick_id, voxel_texture, occupancy_map):
    shared_counts = [0] * 32
    thread_idx = get_local_thread_id()  # 0 to 31
    brick_start = brick_id * 512
    voxel_base = brick_start + (thread_idx * 16)
    count = 0
    for i in range(16):
        voxel_val = voxel_texture[voxel_base + i]
        if voxel_val & 1:  # 1-bit solid/air flag
            count += 1
    shared_counts[thread_idx] = count
    thread_barrier()  # Synchronize all threads in the block
    if thread_idx == 0:
        total = sum(shared_counts)
        occupancy_map[brick_id] = total

# Helper definitions for GPU context
def get_local_thread_id():
    return 0  # Thread 0 to 31 within the block
def thread_barrier():
    pass  # Blocks until all threads in the group reach this point
```

**Edge Cases and Pitfalls:**
*   **Precision:** Uses a 1-bit per voxel solid/air flag; maximum count per brick is 512 [S1, unofficial].
*   **Boundaries:** Assumes brick boundaries align with the voxel grid and texture sampling coordinates [S1, unofficial].
*   **Degenerate Input:** If the voxel texture is not properly bound or aligned, reads may return undefined values [G].

## Subtopics

### [N3-tree with regular mipmapped bricks](6344-n3-tree-with-regular-mipmapped-bricks.md)
A two-level hierarchical data structure combining an N-ary tree for hierarchy with regular voxel bricks in leaves, generalizing octree-based Brick Maps for efficient GPU traversal and filtering. See [N3-tree with regular mipmapped bricks](6344-n3-tree-with-regular-mipmapped-bricks.md).

## Operating characteristics
*   **Time Complexity:** $O(1)$ per brick, proportional to the number of voxels in the brick (512 reads, 32 writes) [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per brick for shared memory (32 integers) [S1, unofficial].
*   **Performance:** Fast due to avoidance of atomic operations and use of fast shared memory [S1, unofficial].
*   **Strengths:** Low latency and no global memory contention [S1, unofficial].
*   **Limitations:** Fixed brick size (512 voxels) and thread count (32) [S1, unofficial].
*   **When to Use:** Interactive voxel simulations where the world changes frequently and collision detection requires up-to-date occupancy data [S1, unofficial].
*   **When Not to Use:** Static volumes or scenes where occupancy rarely changes [G].

## Background
Originated in the context of voxel-based game development, specifically for the "VoxelPlayground" project by Joris Rijsdijk, to handle dynamic voxel changes in a brick-based rendering and physics system [S1, unofficial]. It addresses the problem of maintaining consistency between the voxel data and the hierarchical occupancy map used for efficient traversal and collision detection [S1, unofficial].

## Sources
[S1] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
