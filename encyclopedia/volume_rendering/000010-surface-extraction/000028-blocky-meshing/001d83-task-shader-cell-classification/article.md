# Task Shader Cell Classification

Also called: Task Shader: Cell Classification and Task Generation

## Overview
Task Shader Cell Classification is a GPU-driven pipeline stage that identifies occupied cells within a volume block, compacts their indices, and generates workgroups for a subsequent mesh shader [S1, S2]. It solves the problem of excessive mesh shader invocations on empty or sparse volume data by ensuring that geometry extraction only occurs for cells containing isosurface geometry [S1, S2]. Unlike [Blocky meshing](31-blocky-meshing.md) methods that extract static face sets, this method dynamically partitions work for transient isosurface extraction, acting as a filter that minimizes the number of mesh shader invocations to significantly improve performance [S1, S2].

## How it works
The method processes a volume block of 256 cells using a task shader workgroup [S1, S2]. It classifies cells, compacts their indices into a dense buffer, and partitions them into lists for mesh shader workgroups based on vertex count limits [S1, S2].

### Inputs and Outputs
**Inputs:**
*   Volume scalar field $V$ (accessed via texture/sampler) [G].
*   Block index (32-bit integer) identifying the $256$-cell block to process [S1, S2].
*   Isosurface threshold (implicit in MC classification) [G].

**Outputs (Memory Interface Block):**
*   `dense_occupancy_index`: Buffer containing a compact list of indices of occupied cells within the block [S2].
*   `occupied_cell_list_offset`: Buffer of size 32, storing the start index in `dense_occupancy_index` for each mesh shader workgroup's list [S2].
*   `occupied_list_length`: Buffer of size 32, storing the number of cells in each mesh shader workgroup's list [S2].
*   `gl_TaskCountNV`: Output variable indicating the number of mesh shader workgroups emitted [S1, S2].

### Data Structures
The task shader workgroup uses shared memory for intermediate storage during classification and reduction [S1, S2].

1.  **Occupancy Flags**: A boolean array of size 256 (one per cell in the block) stored in shared memory [S1, S2].
2.  **Vertex Counts**: An integer array of size 256 (one per cell) stored in shared memory, holding the number of vertices required by the MC algorithm for each cell [S1, S2].
3.  **Local Offsets**: An integer array of size 32 (one per thread) used during stream compaction to store the local offset of each thread's first occupied cell within its subgroup of 32 cells [S1, S2].

### Parameters
*   **Block Size**: 256 cells. This allows indexing all cells within a block with a single 8-bit integer, reducing interface block memory overhead [S1, S2].
*   **Workgroup Size**: 32 threads. Each thread processes multiple cells during classification; each mesh shader workgroup processes up to 32 cells [S1, S2].
*   **Max Output Vertices per Mesh Workgroup**: 96. This is the empirically determined locally optimal limit for the test datasets [S1, S2].
*   **Min Cells per Mesh Workgroup**: 8. Ensures the vertex limit is not exceeded even if every cell produces 12 vertices (4 disconnected triangles) ($8 \times 12 = 96$) [S1, S2].
*   **Max Lists per Block**: 32. Calculated as $256 / 8$ [S1, S2].

### Steps

1.  **Individual Cell Classification and Vertex Count Generation** [S1, S2]. The 32-thread workgroup processes the 256 cells [S1, S2]. Each thread handles a subset of cells (8 cells per thread) [S1, S2]. Each thread samples the 8 voxels required by the Marching Cubes (MC) algorithm for its assigned cells [S1, S2]. The thread determines the cell state (occupied/empty) and the number of vertices required for the isosurface [S1, S2]. The task shader **stops** the MC algorithm after determining the vertex count. It does **not** extract primitives or indices [S1, S2]. Occupancy flags and vertex counts are stored in shared memory buffers [S1, S2].

2.  **Stream Compaction (Reduction)** [S1, S2]. A reduction is performed to compact the indices of occupied cells into `dense_occupancy_index` [S1, S2]. The 256 cells are processed in groups of 32 (one per thread) across 8 successive iterations [S1, S2]. Each thread calculates the number of occupied cells in its group [S1, S2]. The last thread in the workgroup shares the total count for its group using `subgroupBroadcast` operations. This avoids expensive atomic operations [S1, S2]. Each thread calculates a global offset for its group's occupied cells [S1, S2]. Threads write the indices of occupied cells into `dense_occupancy_index` at their calculated offsets [S1, S2]. Local offsets are stored in a shared buffer for task generation [S1, S2].

3.  **Task Generation** [S1, S2]. A single thread (e.g., thread 0) traverses the `dense_occupancy_index` buffer to partition occupied cells into lists for mesh shader workgroups [S1, S2]. The thread iterates through the compacted occupied cells [S1, S2]. It accumulates the vertex counts for each cell [S1, S2]. A new list is started when the current list contains 32 cells, or adding the next cell would exceed the 96-vertex limit [S1, S2]. For each completed list, the thread writes the start offset (from `dense_occupancy_index`) to `occupied_cell_list_offset` and the length (number of cells) to `occupied_list_length` [S1, S2]. The thread writes the total number of lists (mesh shader workgroups) to `gl_TaskCountNV` [S1, S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shared memory buffers (declared as comments; actual implementation uses GPU shared memory)
# occupancy_flags: array[256] of bool
# vertex_counts: array[256] of int
# local_offsets: array[32] of int

# Global memory buffers
dense_occupancy_index = []  # Up to 256 entries
occupied_cell_list_offset = [0] * 32
occupied_list_length = [0] * 32

# Constants
BLOCK_SIZE = 256
WORKGROUP_SIZE = 32
MAX_VERTICES_PER_WORKGROUP = 96
MAX_CELLS_PER_WORKGROUP = 32
MIN_CELLS_PER_WORKGROUP = 8

def task_shader(block_index):
    thread_id = get_thread_id()
    
    # Step 1: Individual Cell Classification
    start_cell = thread_id * 8
    end_cell = start_cell + 8
    my_occupied_count = 0
    
    for c in range(start_cell, end_cell):
        voxels = sample_voxels_for_cell(V, c)
        is_occ, num_v = marching_cubes_classify(voxels)
        occupancy_flags[c] = is_occ
        vertex_counts[c] = num_v
        if is_occ:
            my_occupied_count += 1
            
    # Step 2: Stream Compaction (Reduction)
    # [G] Standard warp-level prefix sum and reduction primitives are used here.
    # Compute local offset within the thread's group of 32 cells.
    local_offset = warp_prefix_sum(my_occupied_count)
    
    # Determine global offset for this thread's first occupied cell.
    # [G] Warp reduction to get total occupied count, then prefix sum across threads.
    global_offset_start = warp_prefix_sum(my_occupied_count)
    
    # Write occupied cell indices to dense_occupancy_index
    current_dense_idx = global_offset_start
    for c in range(start_cell, end_cell):
        if occupancy_flags[c]:
            dense_occupancy_index.append(c)
            current_dense_idx += 1
            
    # Store local offset for task generation
    local_offsets[thread_id] = global_offset_start if my_occupied_count > 0 else global_offset_start + my_occupied_count

    # Step 3: Task Generation
    if thread_id == 0:
        num_workgroups = 0
        current_list_start = 0
        current_list_vertex_count = 0
        current_list_cell_count = 0
        
        # [G] Total occupied cells obtained via warp sum reduction
        total_occupied = warp_sum(my_occupied_count)
        
        idx = 0
        while idx < total_occupied:
            cell_index = dense_occupancy_index[idx]
            v_count = vertex_counts[cell_index]
            
            if current_list_cell_count == MAX_CELLS_PER_WORKGROUP or \
               current_list_vertex_count + v_count > MAX_VERTICES_PER_WORKGROUP:
               
               if current_list_cell_count > 0:
                   occupied_cell_list_offset[num_workgroups] = current_list_start
                   occupied_list_length[num_workgroups] = current_list_cell_count
                   num_workgroups += 1
               
               current_list_start = idx
               current_list_vertex_count = 0
               current_list_cell_count = 0
               
            current_list_vertex_count += v_count
            current_list_cell_count += 1
            idx += 1
            
        if current_list_cell_count > 0:
            occupied_cell_list_offset[num_workgroups] = current_list_start
            occupied_list_length[num_workgroups] = current_list_cell_count
            num_workgroups += 1
            
        gl_TaskCountNV = num_workgroups
```

### Edge Cases and Pitfalls
*   **Empty Blocks**: If a block has no occupied cells, `gl_TaskCountNV` should be 0. The task generation logic must handle this gracefully [S1, S2].
*   **Vertex Count Variance**: The number of vertices per cell varies between 3 and 12. The dynamic partitioning ensures the 96-vertex limit is respected [S1, S2].
*   **Memory Efficiency**: Extracting primitives in the task shader would require storing 12 vertices per cell in the interface block, drastically increasing memory usage and reducing performance. Only 8-bit indices are sent [S1, S2].
*   **Atomic Operations**: Expensive atomic operations are omitted by using `subgroupBroadcast` for offset sharing, which is more efficient [S1, S2].

## Subtopics

### [Task Shader Block-wise Stream Compaction](1540-task-shader-block-wise-stream-compaction.md)
A GPU-driven occlusion culling technique that identifies volume blocks containing isosurface geometry and generates proxy geometry to determine the potentially visible set (PVS) of blocks. It solves the problem of high draw times in volumes with high depth complexity by avoiding the extraction and rendering of geometry for blocks that are occluded. Unlike methods that process the depth texture of a previously established occluder representation, such as hierarchical occlusion maps [ZMHH97] or iterative depth warping [LKE18], this approach rasterizes proxy geometry directly. It is a hybrid between basic raster occlusion culling [KT14] and hierarchical approaches [LL20], resolving potential visibility only for fixed-size blocks using a custom task and mesh shader pipeline.

### [Mesh Shader: Transient Isosurface Extraction](1543-mesh-shader-transient-isosurface-extraction.md)
Transient isosurface extraction is a rasterization-based method that extracts isosurface geometry "on the fly" using task and mesh shader pipelines, passing primitives directly to the rasterizer without intermediate explicit mesh storage. It solves the high memory bandwidth and storage costs of indirect methods (like standard Marching Cubes) and the resolution-dependent performance penalties of ray marching. Unlike its parent method, [Task Shader Cell Classification](1535-task-shader-cell-classification.md), which only classifies cells and emits workgroups, this method performs the actual geometry extraction and projection within the mesh shader stage.

## Operating characteristics
*   **Time Complexity**: The classification step is $O(N)$ where $N$ is the number of cells in the block (256). The stream compaction is also $O(N)$ with a small constant factor due to the reduction. Task generation is $O(M)$ where $M$ is the number of occupied cells. Overall, it is linear with respect to the block size [S1, S2].
*   **Memory Complexity**: The interface block memory is fixed per block: `dense_occupancy_index` holds up to 256 cell indices, while `occupied_cell_list_offset` and `occupied_list_length` are fixed at 32 elements each [S1, S2]. Not given in the sources.
*   **Performance**: Minimizes mesh shader invocations, which has a significant positive effect on performance [S1, S2]. The use of `subgroupBroadcast` avoids atomic operations, improving efficiency [S1, S2].
*   **Strengths**: High performance on sparse volumes; efficient use of GPU tree expansion; minimal interface block memory usage [S1, S2].
*   **Limitations**: Requires support for task and mesh shaders (NV/ARB extensions); empirical vertex limit (96) may not be optimal for all hardware or data sets [S1, S2].

## Background
The method was introduced in the paper "Efficient Direct Isosurface Rasterization of Scalar Volumes" [S1, S2]. It leverages the task and mesh shader pipeline features introduced in NVIDIA's RTX architecture to optimize isosurface extraction. The approach builds on the principles of [Blocky meshing](31-blocky-meshing.md) but adapts them for transient, GPU-driven extraction [S1, S2].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
