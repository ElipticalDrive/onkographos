# Dependency Queue Gating

Also called: Dependency queue, Neighbor counter gating, Prerequisite Chunk Loading

## Overview
Dependency Queue Gating delays the mesh generation of a voxel chunk until all its neighboring chunks are loaded, ensuring that complete visibility data is available before culling inner faces [S1, unofficial]. This method sets itself apart from reactive remeshing by proactively managing load order to minimize redundant mesh calculations, trading increased pre-loading checks for reduced re-meshing overhead [S1, unofficial]. It builds on [Asynchronous Chunk Loading](4216-asynchronous-chunk-loading.md) by adding a prerequisite check to the task queue.

## How it works
**Inputs and Outputs:**
- Input: A request to generate a mesh for a target chunk $C_{target}$ at coordinates $(x, y, z)$.
- Output: A completed mesh for $C_{target}$, generated only after neighbors are ready.

**Data Structures:**
- `ChunkState`: An enumeration or bitmask tracking the status of each chunk (e.g., `Unloaded`, `Loading`, `Loaded`, `MeshGenerated`).
- `NeighborCounter`: An integer associated with each chunk, counting the number of loaded immediate neighbors.
- `World`: A logical representation providing access to any block by coordinates, returning block ID or 0 for air, or -2 for out-of-bounds [S1, unofficial].

**Steps:**
1. **Initialize Counter:** When a chunk $C$ is requested for meshing, initialize a counter $N_{loaded} = 0$.
2. **Check Neighbors:** Identify the 6 immediate neighbors of $C$ in 3D space (or 4 in 2D, though 3D is implied by the threshold of 6) [S1, unofficial].
3. **Load Prerequisites:** For each neighbor $N_i$:
   - If $N_i$ is not in `Loaded` state, trigger its asynchronous loading process.
   - Wait for $N_i$ to reach `Loaded` state.
   - Increment $N_{loaded}$ upon confirmation of $N_i$'s load completion.
4. **Threshold Check:** Continue until $N_{loaded}$ reaches the threshold of 6 (for 3D adjacency) [S1, unofficial].
5. **Generate Mesh:** Once the threshold is met, proceed to generate the mesh for $C_{target}$. During this step, use the block data from the loaded neighbors to correctly cull inner faces at the chunk boundaries [S1, unofficial].
6. **Propagate Lighting:** If the engine uses lighting propagation, propagate lighting data from the newly generated chunk to its neighbors if necessary, as this step cannot be fully avoided by gating alone [S1, unofficial].

**Parameters:**
- **Neighbor Load Threshold:** 6 for 3D voxel chunks (representing the 6 faces of a cube) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Lighting Propagation:** While gating reduces mesh recalculation, it does not eliminate the need for lighting propagation, which may still require updates when neighbors change [S1, unofficial].
- **Pre-loading Overhead:** The method increases the number of pre-loading checks and dependencies, potentially creating bottlenecks if the thread pool is saturated [S1, unofficial].
- **Boundary Conditions:** The world function must correctly handle out-of-bounds checks (returning -2) to prevent errors when checking neighbors at the edge of the world [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk_mesh(chunk_coords, world, chunk_states, neighbor_threshold=6):
    """
    Generates mesh for a chunk only after all neighbors are loaded.
    """
    x, y, z = chunk_coords
    
    # Define 6 neighbors in 3D space
    neighbors = [
        (x+1, y, z), (x-1, y, z),
        (x, y+1, z), (x, y-1, z),
        (x, y, z+1), (x, y, z-1)
    ]
    
    loaded_count = 0
    
    # Step 1 & 2: Check and load prerequisites
    for n_coords in neighbors:
        # Check if neighbor is already loaded
        if chunk_states.get(n_coords) == 'Loaded':
            loaded_count += 1
            continue
            
        # If not loaded, trigger async load and wait
        # Note: In a real async system, this would involve futures/callbacks
        # Here we simulate the blocking wait for clarity of logic
        load_chunk_async(n_coords, world, chunk_states)
        wait_for_chunk(n_coords, chunk_states)
        
        if chunk_states.get(n_coords) == 'Loaded':
            loaded_count += 1
            
    # Step 3: Threshold check
    if loaded_count < neighbor_threshold:
        # This case should ideally not happen if logic is correct,
        # but serves as a guard.
        raise Exception("Not all neighbors loaded")
        
    # Step 4: Generate mesh
    mesh = build_mesh(chunk_coords, world, chunk_states)
    
    # Step 5: Propagate lighting if necessary
    propagate_lighting(chunk_coords, world, chunk_states)
    
    chunk_states[chunk_coords] = 'MeshGenerated'
    return mesh
```

## Subtopics

### Spiral Dependency Queue
Queues chunk generation and tessellation tasks in a spiral pattern from the player's position, ensuring neighbors are generated before their adjacent chunks are tessellated, guaranteeing edge visibility [S1, unofficial]. It differs from standard gating by enforcing a strict sequential order via queue depth rather than just a counter, allowing single-threaded generation/tessellation without race conditions [S1, unofficial]. Choose this when you want to avoid multi-threading complexity in mesh building while ensuring correct edge culling [S1, unofficial]. Pitfall: Complex to implement and describe; requires careful queue management to guarantee neighbor state before tessellation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Increases pre-loading checks. The time to generate a chunk is now dependent on the time to load all neighbors.
- **Memory Complexity:** Requires storing `ChunkState` for all chunks in the vicinity.
- **Measured Performance:** Significantly reduces re-meshing overhead by avoiding redundant calculations when neighbors load in [S1, unofficial].
- **Strengths:** Ensures complete visibility data for culling; minimizes redundant mesh calculations.
- **Limitations:** Increases pre-loading checks; requires lighting propagation; complex to implement in spiral variant.
- **When to Use:** In voxel game engines where chunk generation order is critical and redundant meshing is costly.
- **When Not to Use:** When lighting propagation is expensive and cannot be optimized, or when strict single-threaded simplicity is not required and parallel meshing is preferred.

## Background
Originating from community discussions in voxel game development, this method was proposed as a solution to the problem of culling inner faces when neighboring chunks are not yet loaded [S1, unofficial]. It was developed by developers such as team23, DubstepCoder, and YamBazi in 2014, who shared implementations and strategies for managing chunk dependencies in voxel engines [S1, unofficial]. It is used in voxel game engines to manage chunk generation order and minimize redundant mesh calculations [S1, unofficial].

## Sources
[S1] Tips for culling inner faces? : r/VoxelGameDev, https://www.reddit.com/r/VoxelGameDev/comments/210u58/tips_for_culling_inner_faces/, fetched 2026-10-27 (unofficial)
