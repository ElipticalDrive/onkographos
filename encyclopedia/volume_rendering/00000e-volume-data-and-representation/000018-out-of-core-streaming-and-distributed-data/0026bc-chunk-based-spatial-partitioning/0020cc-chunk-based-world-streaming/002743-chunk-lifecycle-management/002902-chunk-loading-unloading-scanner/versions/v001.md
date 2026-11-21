# Chunk Loading/Unloading Scanner

Also called: Scanner, ScannerPlugin

## Overview
The Chunk Loading/Unloading Scanner identifies which chunks in a volumetric world need to be loaded or unloaded based on the camera's current and previous positions, using offset grids to define load and unload areas [S1, unofficial]. It differs from [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) by explicitly tracking state transitions between frames via previous position comparison, and from [Budget Plans](5497-budget-plans.md) by using a fixed-radius offset strategy rather than a dynamic priority queue [S1, unofficial].

## How it works
**Inputs:** Camera transform (translation), previous chunk position, chunk size (32 units), and sampling offsets.
**Outputs:** Lists of chunks to load (`data_load`, `mesh_load`) and unload (`data_unload`, `mesh_unload`).

**Data Structures:**
The `Scanner` struct maintains the following fields [S1, unofficial]:
- `prev_chunk_pos`: `IVec3` (previous frame's chunk coordinates).
- `checks_per_frame`: `usize` (limit on chunks processed per frame).
- `data_offset`: `usize` (radius for data loading).
- `mesh_offset`: `usize` (radius for mesh loading).
- `unresolved_data_load`: `Vec<IVec3>` (chunks pending data load).
- `unresolved_mesh_load`: `Vec<IVec3>` (chunks pending mesh load).
- `unresolved_data_unload`: `VecDeque<IVec3>` (chunks pending data unload).
- `unresolved_mesh_unload`: `VecDeque<IVec3>` (chunks pending mesh unload).
- `data_sampling_offsets`: `Vec<IVec3>` (precomputed offsets for data).
- `mesh_sampling_offsets`: `Vec<IVec3>` (precomputed offsets for mesh).

**Steps:**
1. **Calculate Current Chunk Position:** Convert the camera's world translation to chunk coordinates. Given a chunk size of 32 and an offset of 16 (half-chunk), the position is calculated as [S1, unofficial]:
   $$ \text{chunk\_pos} = \left\lfloor \frac{\text{translation} - 16.0}{32.0} \right\rfloor $$
   In integer arithmetic:
   ```python
   chunk_pos = ((camera_translation - 16.0) / 32.0).as_ivec3()
   ```

2. **Check for Movement:** Compare `chunk_pos` with `prev_chunk_pos`. If they are identical, skip processing for this frame [S1, unofficial].

3. **Generate Load/Unload Areas:** Create sets of chunk coordinates to consider for loading and unloading by adding sampling offsets to the current and previous positions, respectively [S1, unofficial]:
   $$ \text{load\_data\_area} = \text{chunk\_pos} + \text{data\_sampling\_offsets} $$
   $$ \text{unload\_data\_area} = \text{prev\_chunk\_pos} + \text{data\_sampling\_offsets} $$
   Similar areas are generated for mesh loading using `mesh_sampling_offsets`.

4. **Compute Differences:** Determine the symmetric difference between the current load area and the previous unload area to identify chunks that need to be loaded or unloaded [S1, unofficial]:
   $$ \text{data\_load} = \text{load\_data\_area} \setminus \text{unload\_data\_area} $$
   $$ \text{data\_unload} = \text{unload\_data\_area} \setminus \text{load\_data\_area} $$
   $$ \text{mesh\_load} = \text{load\_mesh\_area} \setminus \text{unload\_mesh\_area} $$
   $$ \text{mesh\_unload} = \text{unload\_mesh\_area} \setminus \text{load\_mesh\_area} $$

5. **Update Queues:**
   - Add chunks from `data_load` and `mesh_load` to their respective unresolved queues.
   - Remove chunks from `data_load` and `mesh_load` from the unload queues (if present).
   - Remove chunks from `data_unload` and `mesh_unload` from the load queues (if present).

6. **Sort Load Queues:** Sort `unresolved_data_load` and `unresolved_mesh_load` by squared distance to the current `chunk_pos` to prioritize nearby chunks [S1, unofficial].

7. **Update State:** Set `prev_chunk_pos = chunk_pos` for the next frame.

**Parameters:**
- `checks_per_frame`: Default value is $32 \times 32 \times 32$ [S1, unofficial].
- `data_distance`: Radius for data loading offsets, calculated as `distance + 1` [S1, unofficial].
- `mesh_distance`: Radius for mesh loading offsets, calculated as `distance` [S1, unofficial].

**Edge Cases:**
- If the camera does not move between frames, no chunks are loaded or unloaded [S1, unofficial].
- Precision issues in chunk position calculation can cause flickering if the camera is near chunk boundaries; using floor division with an offset (16.0) helps center the active chunk [S1, unofficial].

## Subtopics

### [Chunk Visibility Culling](5031-chunk-visibility-culling.md)
A technique to determine which chunks are visible to the camera by checking if their bounding boxes intersect the view frustum, sorted by distance.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of chunks in the sampling offset area. Sorting adds $O(K \log K)$ where $K$ is the number of unresolved loads.
- **Memory Complexity:** Proportional to the number of unresolved chunks in the queues.
- **Strengths:** Simple to implement, deterministic behavior based on camera movement, effective for large worlds with limited memory.
- **Limitations:** May load unnecessary chunks if the camera moves erratically; does not account for occlusion or view frustum culling directly.
- **When to use:** When memory is constrained and the world is too large to fit in RAM.
- **When not to use:** When the world fits in memory or when occlusion culling is more critical than streaming.

## Background
The Chunk Loading/Unloading Scanner is part of the Binary Greedy Voxel Meshing Algorithm implementation, specifically designed for out-of-core streaming of voxel data [S1, unofficial]. It was developed to manage memory usage in large-scale voxel environments by dynamically loading and unloading chunks based on camera proximity.

## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
