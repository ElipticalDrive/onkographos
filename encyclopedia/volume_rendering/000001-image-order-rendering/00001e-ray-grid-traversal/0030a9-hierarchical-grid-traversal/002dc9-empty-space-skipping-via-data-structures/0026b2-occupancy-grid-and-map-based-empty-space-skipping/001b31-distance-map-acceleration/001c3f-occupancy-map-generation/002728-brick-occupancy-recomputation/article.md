# Brick Occupancy Recomputation
Also called: Brick Occupancy Compute
## Overview
Brick Occupancy Recomputation is a GPU-based method for updating the occupancy status of volumetric bricks after dynamic changes to the underlying voxel data. It solves the problem of stale occupancy maps in interactive voxel environments where the world state changes frequently, such as in voxel-based games or simulations. Unlike static generation methods, this approach recomputes occupancy per brick using a lightweight compute shader, avoiding atomic operations by leveraging shared memory for intra-brick reduction. It builds on [Occupancy Map Generation](1197-occupancy-map-generation.md) but is specifically optimized for incremental updates in dynamic scenes.
## How it works
**Inputs and Outputs:**
- Input: A texture or buffer containing the current voxel data for a specific brick.
- Output: A single unsigned integer representing the occupancy count for that brick, written to a global "brickmap" or occupancy texture.

**Data Structures:**
- **Shared Memory Array:** An array of 32 unsigned integers (`uint`), one per thread in the block.
- **Voxel Data:** Read from global memory. The source specifies reading 16 voxels per thread.
- **Brickmap Entry:** A single `uint` in global memory corresponding to the brick being processed.

**Parameters:**
- **Threads per Block:** Exactly 32.
- **Voxels per Thread:** 16.
- **Total Voxels per Brick:** $32 \times 16 = 512$ voxels.
- **Air Threshold:** Voxels are considered "non-air" if they are not air. The exact definition of "air" depends on the voxel format (e.g., density < threshold or specific material ID).

**Steps:**
1. **Thread Identification:** Each thread in the 32-thread block identifies its unique index `tid` (0 to 31).
2. **Voxel Reading:** Each thread reads 16 voxels from its assigned portion of the brick's voxel data. The mapping of threads to voxel indices is not explicitly detailed, but it implies a partitioning of the 512 voxels among the 32 threads.
3. **Counting:** Each thread counts the number of non-air voxels in its 16-voxel chunk. Let this count be $c_{tid}$.
4. **Shared Memory Write:** Each thread writes $c_{tid}$ to `shared_counts[tid]` in shared memory.
5. **Synchronization:** All threads in the block execute a barrier synchronization (`__syncthreads()`) to ensure all counts are written.
6. **Reduction:** The first thread (`tid == 0`) sums all values in `shared_counts` to compute the total occupancy count $C = \sum_{tid=0}^{31} c_{tid}$.
7. **Global Write:** Thread 0 writes $C$ to the brickmap entry for the current brick.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume this code runs in a compute shader dispatch where one block handles one brick.
# shared_counts is a shared memory array of 32 uints.
# voxel_data is a global memory buffer containing 512 voxels for this brick.
# brickmap is a global memory buffer for occupancy counts.

tid = get_thread_id_in_block() # 0 to 31
local_count = 0

# Each thread reads 16 voxels. The exact indexing scheme is not specified,
# but it partitions the 512 voxels.
# Let's assume a simple row-major or linear partitioning for implementation.
# Total voxels = 512. Voxels per thread = 16.
voxels_per_thread = 16
start_voxel_index = tid * voxels_per_thread

for i in range(voxels_per_thread):
    voxel_index = start_voxel_index + i
    voxel = read_voxel_from_global_memory(voxel_data, voxel_index)
    if not is_air(voxel): # Implementation of is_air depends on voxel format
        local_count += 1

shared_counts[tid] = local_count
sync_threads_in_block() # __syncthreads()

if tid == 0:
    total_occupancy = 0
    for i in range(32):
        total_occupancy += shared_counts[i]
    write_to_brickmap(brickmap, brick_id, total_occupancy)
```

**Edge Cases and Pitfalls:**
- **Precision:** The occupancy count is an unsigned integer. For a 512-voxel brick, the maximum count is 512, which fits in a standard `uint`.
- **Boundaries:** The method assumes the brick data is fully available in global memory.
- **Degenerate Input:** If the voxel data is not properly loaded or the brick ID is invalid, the write to the brickmap may be incorrect.

## Subtopics

### [N3-tree with regular mipmapped bricks](6344-n3-tree-with-regular-mipmapped-bricks.md)
A two-level hierarchical structure for procedural volume exploration combining an N3-tree (an N-ary tree with N^3 children) for hierarchy and regular voxel bricks stored in the leaves. It links the compactness of a hierarchical structure with the efficient traversal and trilinear/quadrilinear filtering capabilities of regular grids on GPU.

## Operating characteristics
- **Time Complexity:** $O(1)$ per brick, as the number of threads and voxels per thread is fixed.
- **Memory Complexity:** $O(1)$ per brick for shared memory (32 `uint`s).
- **Performance:** Fast, as it avoids atomic operations and uses shared memory for reduction.
- **Strengths:** Low latency, no contention on global memory for reduction, suitable for dynamic scenes.
- **Limitations:** Fixed brick size (512 voxels), requires a compute shader.
- **When to use:** Dynamic voxel environments where occupancy needs frequent updates.
- **When not to use:** Static volumes where occupancy is computed once.

## Background
Originated in the context of voxel-based game development, specifically for handling dynamic voxel worlds where players can modify the terrain. The method was described by Joris Rijsdijk in an unofficial blog post discussing voxel simulation and occupancy map updates.

## Sources
[S1] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
