# Mesh Shader: Transient Isosurface Extraction

Also called: Direct Isosurface Rendering, Direct Isosurface Visualization, Efficient Direct Isosurface Rasterization, Mesh Shader Isosurface Extraction, Task and Mesh Shader Isosurface Extraction, Transient Isosurface Extraction, Transient Isosurface Extraction and Rendering, Transient Isosurface Extraction and Rendering of PVS

## Overview
Transient isosurface extraction is a rasterization-based method that extracts isosurface geometry "on the fly" using task and mesh shader pipelines, passing primitives directly to the rasterizer without intermediate explicit mesh storage [S1][S3, unofficial]. It solves the high memory bandwidth and storage costs of indirect methods (like standard Marching Cubes) and the resolution-dependent performance penalties of ray marching [S1][S3, unofficial]. Unlike its parent method, [Task Shader Cell Classification](1535-task-shader-cell-classification.md), which only classifies cells and emits workgroups, this method performs the actual geometry extraction and projection within the mesh shader stage [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A dense scalar volume, a min-max grid (2-channel volume texture storing min/max values per block), and a Potentially Visible Set (PVS) of block indices [S1][S3, unofficial].
*   **Output:** Triangle primitives passed directly to the rasterizer [S1][S3, unofficial].

**Data Structures:**
The method processes volume blocks of size $U \times V \times W$ cells. The standard setting is $U=8, V=8, W=4$, totaling 256 cells per block, allowing 8-bit indexing [S3, unofficial].

The memory interface block between the task shader and mesh shader is defined as [S3, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Task {
    uint32_t baseID;
    uint8_t dense_occupancy_index[256];
    uint8_t occupied_cell_list_offset[32];
    uint8_t occupied_list_length[32];
} INTERFACE_TASK_MESH_EXTRACT_AND_DRAW;
```
*   `baseID`: 32-bit index of the block [S3, unofficial].
*   `dense_occupancy_index`: Compact list of occupied cell indices within the block [S3, unofficial].
*   `occupied_cell_list_offset`: Start offset for each mesh shader workgroup's cell list [S3, unofficial].
*   `occupied_list_length`: Length of each mesh shader workgroup's cell list [S3, unofficial].

**Parameters:**
*   **Workgroup Size:** 32 threads for both task and mesh shaders [S3, unofficial].
*   **Vertex Limit:** Maximum 96 vertices per mesh shader workgroup [S3, unofficial].
*   **Min Cells per Workgroup:** 8 cells (ensures $8 \times 12$ vertices $\le$ 96, assuming max 12 vertices per cell) [S3, unofficial].
*   **Max Lists per Block:** 32 (calculated as $256 / 8$) [S3, unofficial].

**Steps:**

1.  **Task Shader: Cell Classification and Vertex Count Generation**
    The task shader workgroup (32 threads) processes a block of 256 cells. Each thread handles multiple cells [S3, unofficial].
    *   Threads sample voxels to determine the state of their assigned cells using the Marching Cubes (MC) algorithm [S3, unofficial].
    *   Threads store occupancy flags and write the **vertex count** (not the geometry) for each cell into a shared memory buffer [S3, unofficial].
    *   The MC algorithm stops after determining the per-cell vertex count to minimize interface block memory usage [S3, unofficial].

2.  **Task Shader: Stream Compaction and Task Generation**
    *   **Reduction:** An intra-warp reduction determines the offset of each occupied cell in the `dense_occupancy_index` buffer. Expensive atomic operations are omitted; offsets are shared via `subgroupBroadcast` operations [S3, unofficial].
    *   **List Division:** A single thread traverses the `dense_occupancy_index` buffer. It accumulates vertex counts to build lists for mesh shader workgroups. A list ends when it contains 32 cells OR the accumulated vertex count would exceed 96 [S3, unofficial].
    *   **Buffer Update:** The start offset and length for each list are stored in `occupied_cell_list_offset` and `occupied_list_length` [S3, unofficial].
    *   **Task Count:** One thread writes the total number of mesh shader workgroups to `gl_TaskCountNV` [S3, unofficial].

3.  **Mesh Shader: Lookup**
    Each mesh shader workgroup (32 threads) uses `gl_WorkGroupID` to look up the start offset and length of its specific cell list from the interface buffers [S1][S3, unofficial].

4.  **Mesh Shader: Extraction and Projection**
    *   Each active thread determines its cell ID and location [S3, unofficial].
    *   The thread performs the full MC algorithm to extract vertices and indices representing the isosurface [S3, unofficial].
    *   **Intra-warp Reduction:** An efficient intra-warp reduction determines the write offsets for each thread to write indices and projected vertices into output arrays [S3, unofficial].
    *   **Projection:** Generated vertices are evenly distributed across the warp and projected cooperatively by all threads [S3, unofficial].
    *   **Primitive Count:** One designated thread writes the total primitive count produced by the workgroup [S3, unofficial].
    *   The primitives are passed directly to the rasterizer [S1][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Task Shader Workgroup (32 threads)
def task_shader_workgroup(block_id, volume, min_max_grid):
    # 1. Cell Classification and Vertex Count
    # Each thread handles multiple cells in the 256-cell block
    shared_vertex_counts = [0] * 256
    
    for cell_idx in range(256):
        if cell_idx % 32 == thread_id: # Simplified distribution
            # Sample voxels and determine MC state
            state = get_mc_state(cell_idx, volume)
            if state != EMPTY:
                # Store occupancy
                dense_occupancy_index[cell_idx] = cell_idx
                # Compute vertex count only
                v_count = get_mc_vertex_count(state)
                shared_vertex_counts[cell_idx] = v_count
            else:
                dense_occupancy_index[cell_idx] = 255 # Marker for empty

    # 2. Stream Compaction (Reduction)
    # Calculate offsets for occupied cells in dense_occupancy_index
    # Using subgroupBroadcast to avoid atomics
    occupied_offsets = intra_warp_reduction(dense_occupancy_index, shared_vertex_counts)

    # 3. List Division
    # Single thread divides cells into lists for mesh shader workgroups
    if thread_id == 0:
        list_count = 0
        current_offset = 0
        current_vertex_sum = 0
        current_cell_count = 0
        
        for i in range(256):
            if dense_occupancy_index[i] != 255:
                v = shared_vertex_counts[i]
                if current_cell_count == 32 or current_vertex_sum + v > 96:
                    # End current list
                    occupied_cell_list_offset[list_count] = current_offset
                    occupied_list_length[list_count] = current_cell_count
                    list_count += 1
                    current_offset = i
                    current_vertex_sum = 0
                    current_cell_count = 0
                
                current_vertex_sum += v
                current_cell_count += 1
        
        # Handle last list
        if current_cell_count > 0:
            occupied_cell_list_offset[list_count] = current_offset
            occupied_list_length[list_count] = current_cell_count
            list_count += 1
            
        gl_TaskCountNV = list_count

# Mesh Shader Workgroup (32 threads)
def mesh_shader_workgroup(workgroup_id, interface_block):
    # 1. Lookup
    offset = interface_block.occupied_cell_list_offset[workgroup_id]
    length = interface_block.occupied_list_length[workgroup_id]
    
    # 2. Extraction and Projection
    # Determine write offsets via intra-warp reduction
    write_offsets = intra_warp_reduction(length)
    
    for i in range(length):
        cell_idx = interface_block.dense_occupancy_index[offset + i]
        
        # Extract vertices and indices
        vertices, indices = extract_mc_geometry(cell_idx)
        
        # Project vertices
        projected_vertices = project(vertices)
        
        # Write to output buffers using cooperative projection and offsets
        write_output(write_offsets[i], projected_vertices, indices)
        
    # 3. Primitive Count
    if thread_id == 0:
        write_primitive_count(total_primitives_generated)
```

## Subtopics

### [Poisson Meshlet-Based (PMB) Extraction](1860-poisson-meshlet-based-pmb-extraction.md)
A variant that partitions volumes into larger blocks and extracts persistent meshlets, using hierarchical subblocks to respect meshlet size constraints. Choose when persistent geometry buffers are preferred over transient extraction.

### [Direct Isosurface Rasterization](1894-direct-isosurface-rasterization.md)
A rasterization-based approach that avoids intermediate explicit mesh storage by extracting primitives on the fly. Choose when minimizing memory bandwidth and storage is critical compared to ray-marching.

## Operating characteristics
*   **Time Complexity:** The method's cost is less dependent on rendering resolution compared to ray marching, as rasterization hardware is highly optimized [S1][S3, unofficial].
*   **Measured Performance:** At 3840 × 2160 pixels, direct isosurface rasterization outperformed min-max octree ray marching in all tested scenes: CT Head (0.19ms vs 1.54ms), Vertebra (0.26ms vs 1.95ms), Chameleon (1.26ms vs 2.59ms), and Timeseries Supernova (0.29ms vs 1.41ms) [S1].
*   **Memory:** Requires negligible additional memory and little preprocessing, avoiding the significant memory bandwidth overhead of storing explicit isosurface representations [S1][S3, unofficial].
*   **Strengths:** Suitable for interactive analysis with frequent isovalue adjustments; integrates easily into existing rasterization pipelines; supports multi-view stereo [S1].
*   **Limitations:** Vertex duplication occurs because cells are processed independently; unique vertex extraction is omitted to avoid branch divergence in sparse cells [S1][S3, unofficial]. Task and mesh shaders have limited output sizes compared to compute shaders [S3, unofficial].

## Background
The method was proposed in "Efficient Direct Isosurface Rasterization of Scalar Volumes" (2023) [S1]. It leverages the NVIDIA task and mesh shader pipelines (OpenGL API terminology) to extract and rasterize geometry on the fly [S1]. The approach builds on raster occlusion culling techniques [KT14, LJSL21] to identify potentially visible blocks (PVS) [S1]. It uses the Marching Cubes cell classification [LC87] for geometry extraction.

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
