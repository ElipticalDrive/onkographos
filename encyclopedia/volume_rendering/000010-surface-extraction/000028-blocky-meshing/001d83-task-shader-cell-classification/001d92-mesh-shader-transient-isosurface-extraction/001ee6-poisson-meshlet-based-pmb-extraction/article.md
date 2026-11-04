# Poisson Meshlet-Based (PMB) Extraction

Also called: Meshlet-Based Isosurface Extraction, Real-Time Meshlet Extraction from Scalar Volumes

## Overview
Poisson Meshlet-Based (PMB) Extraction is a real-time isosurface extraction method that partitions scalar volumes into larger blocks and extracts non-trivial persistent meshlets compatible with mesh shader pipelines, improving rendering efficiency over trivial small-voxel approaches [S1, unofficial]. It differs from its parent method, [Mesh Shader: Transient Isosurface Extraction](1543-mesh-shader-transient-isosurface-extraction.md), by introducing a hierarchical partitioning of extraction blocks into subblocks of varying sizes (8x8x4, 4x4x4, 2x2x2) to respect meshlet size constraints while minimizing overhead [S1, unofficial]. Unlike [Direct Isosurface Rasterization](1894-direct-isosurface-rasterization.md), which rasterizes directly, PMB generates persistent geometry buffers for subsequent rendering [S1, unofficial].

## How it works
The method uses a two-stage GPU pipeline: a region analysis task shader stage followed by a meshlet extraction mesh shader stage [S1, unofficial].

### Inputs and Outputs
*   **Input:** A 3D scalar volume dataset, an isovalue threshold, and rendering parameters (viewing frustum for culling).
*   **Output:** Global vertex and index buffers containing meshlet geometry, and a dense array of meshlet descriptors containing offsets to buffer segments [S1, unofficial].

### Data Structures
*   **Extraction Block:** A 3D grid of cells with dimensions $8 \times 8 \times 4$ (256 cells total) [S1, unofficial].
*   **Subblocks:** Hierarchical partitions of the extraction block: $8 \times 8 \times 4$, $4 \times 4 \times 4$, and $2 \times 2 \times 2$ [S1, unofficial].
*   **Memory Interface:** On-chip memory passed from task shader to mesh shader. It encodes:
    *   All occupied cell indices within a subblock.
    *   Offset to the first occupied cell.
    *   Number of occupied cells for each subblock [S1, unofficial].
*   **Meshlet Descriptor:** Stored in a dense global array. Contains:
    *   Index of the first meshlet extracted from the block.
    *   Number of meshlets the block contains [S1, unofficial].
*   **Global Buffers:**
    *   Vertex buffer: Stores unique vertex positions and attributes.
    *   Index buffer: Stores triangle indices.
    *   Descriptor array: Aligned with occupied block indices [S1, unofficial].

### Parameters
*   $T$: Workgroup size (number of threads). Typical value: 32 (standard subgroup size on NVIDIA GPUs) [S1, unofficial].
*   $R$: Number of cells analyzed by each thread. Typical value: 8 [S1, unofficial].
*   $\beta$: Number of cells in an extraction block. Calculated as $T \times R$. Value: 256. Block dimensions: $8 \times 8 \times 4$ cells [S1, unofficial].
*   $C$: Maximum number of occupied cells permitted within a subblock. Typical value: 128. Limits memory interface size and work per thread [S1, unofficial].

### Steps

1.  **Region Analysis (Task Shader):**
    1.  The task shader workgroup processes the extraction block. Each thread analyzes $R=8$ cells [S1, unofficial].
    2.  For each cell, determine if it is a **core** or **padding** cell for each possible subblock size ($8 \times 8 \times 4$, $4 \times 4 \times 4$, $2 \times 2 \times 2$) using modulo operations based on the cell's position within the block [S1, unofficial].
    3.  Characterize isosurface geometry at the subblock level to derive subregions whose meshlets approach optimal vertex and index counts while satisfying strict meshlet size constraints [S1, unofficial].
    4.  Calculate the number of vertices and indices created by the block.
    5.  Hierarchically partition the block into subblocks ($8 \times 8$, $4 \times 4$, $2 \times 2$ in 2D projection, corresponding to 3D volumes) whose extracted geometry respects meshlet size limits while avoiding arbitrarily small meshlets [S1, unofficial].
    6.  Reserve space in dense global memory buffers for meshlet descriptors, indices, and vertex attributes [S1, unofficial].
    7.  Store the index of the first meshlet extracted from the block and the number of meshlets the block contains in a dense array aligned with occupied block indices [S1, unofficial].
    8.  Encode all occupied cell indices in the on-chip memory interface passed to mesh shaders, along with an offset to the first occupied cell and number of occupied cells for each subblock [S1, unofficial].

2.  **Meshlet Extraction (Mesh Shader):**
    1.  Each mesh shader workgroup extracts exactly one meshlet from its assigned subblock [S1, unofficial].
    2.  Read data from the task and mesh memory interface to determine which parent block cells to use [S1, unofficial].
    3.  Extend the Parallel Marching Blocks (PMB) technique [LCDW16] by separating vertex extraction and index extraction stages:
        *   First, extract vertices and vertex indices from all occupied cells. Allow each thread to process multiple cells (up to $C=128$ occupied cells per subblock) to reduce the number of mesh shader workgroups [S1, unofficial].
        *   Then, derive triangle indices from the extracted vertices [S1, unofficial].
    4.  Write vertex and index data to global vertex and index buffers [S1, unofficial].
    5.  Store a meshlet descriptor containing offsets to the relevant buffer segments [S1, unofficial].
    6.  **Culling (Optional):** Extract bounding volume representations (implicit AABBs, explicit AABBs, bounding spheres, normal cones) during meshlet extraction for rendering culling [S1, unofficial].

3.  **Rendering (Task Shader - Optional Culling):**
    1.  If culling is active, launch task shader workgroups so each thread checks if one meshlet should be culled with respect to the viewing frustum [S1, unofficial].
    2.  Store global indices of potentially visible meshlets in the on-chip memory interface [S1, unofficial].
    3.  Launch a mesh shader workgroup for each potentially visible meshlet [S1, unofficial].

4.  **Rendering (Mesh Shader):**
    1.  Retrieve geometry buffer offsets and counts based on local meshlet descriptors [S1, unofficial].
    2.  Read, project, and write meshlet vertex information to a local vertex buffer [S1, unofficial].
    3.  Transfer index data to a local index buffer [S1, unofficial].
    4.  Rasterize and fragment shade [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Task Shader: Region Analysis
def task_shader_region_analysis(block_id):
    # Each thread analyzes R=8 cells
    occupied_cells = []
    subblock_partitions = []
    
    for thread_idx in range(T): # T=32
        for cell_offset in range(R): # R=8
            cell_pos = get_cell_position(block_id, thread_idx, cell_offset)
            if is_occupied(cell_pos, isovalue):
                occupied_cells.append(cell_pos)
                
                # Determine core/padding status for each subblock size
                # Using modulo operations based on cell position
                is_core_8x8x4 = is_core_cell(cell_pos, 8, 8, 4)
                is_core_4x4x4 = is_core_cell(cell_pos, 4, 4, 4)
                is_core_2x2x2 = is_core_cell(cell_pos, 2, 2, 2)
                
                # Accumulate vertex/index counts for potential subblocks
                # ... (logic to determine optimal partitioning)
                
    # Hierarchical partitioning
    subblocks = partition_block(occupied_cells, max_occupied_per_subblock=C)
    
    # Reserve space in global buffers
    descriptor_offset = reserve_global_memory(len(subblocks))
    
    # Encode occupied cells and offsets for mesh shader
    memory_interface = encode_memory_interface(subblocks, occupied_cells)
    
    # Store descriptor info
    write_descriptor(block_id, descriptor_offset, len(subblocks))
    
    return memory_interface

# Mesh Shader: Meshlet Extraction
def mesh_shader_extraction(subblock_id, memory_interface):
    # Read occupied cells for this subblock from memory interface
    occupied_cells = decode_occupied_cells(memory_interface, subblock_id)
    
    # Stage 1: Vertex Extraction
    # Each thread may process multiple cells (up to C=128 total)
    vertices = []
    vertex_indices = []
    
    for cell in occupied_cells:
        # Extract vertex positions and attributes using PMB-style logic
        # to avoid duplicate vertices within the meshlet
        v = extract_vertex(cell)
        idx = add_vertex(vertices, v)
        vertex_indices.append(idx)
        
    # Stage 2: Index Extraction
    # Derive triangle indices from vertex indices
    indices = extract_triangles(vertex_indices)
    
    # Write to global buffers
    global_vertex_buffer.write(vertices)
    global_index_buffer.write(indices)
    
    # Store descriptor
    descriptor = {
        'vertex_offset': global_vertex_buffer.current_offset(),
        'index_offset': global_index_buffer.current_offset(),
        'vertex_count': len(vertices),
        'index_count': len(indices)
    }
    write_meshlet_descriptor(descriptor)
    
    # Optional: Compute bounding volumes
    if culling_enabled:
        compute_bounding_volumes(vertices)

# Helper: Determine if cell is core or padding
def is_core_cell(cell_pos, dim_x, dim_y, dim_z):
    # Core cells are those not on the boundary of the subblock
    # Padding cells are on the boundary and needed for connectivity
    # Using modulo operations
    x, y, z = cell_pos
    is_core_x = (x % dim_x) != 0 and (x % dim_x) != (dim_x - 1)
    is_core_y = (y % dim_y) != 0 and (y % dim_y) != (dim_y - 1)
    is_core_z = (z % dim_z) != 0 and (z % dim_z) != (dim_z - 1)
    return is_core_x and is_core_y and is_core_z
```

### Edge Cases and Pitfalls
*   **Workgroup Size:** Increasing workgroup size beyond a single subgroup ($T > 32$) causes performance degradation due to overhead of communicating between subgroups (shared memory and explicit barriers) [S1, unofficial]. Vulkan implementations exhibit performance degradations when employing workgroups consisting of more than one warp ($T > 32$) [S1, unofficial].
*   **Culling Overhead:** Computing normal cones for culling imposes high extraction overhead and is unsuitable because geometry within a meshlet can exhibit arbitrary normal spread [S1, unofficial]. Implicit AABBs and explicit AABBs provide the best tradeoff [S1, unofficial].
*   **Meshlet Size Limits:** The smallest subblock size ($2 \times 2 \times 2$) results in a maximum of 54 vertices and 40 primitives, which does not exceed typical size limits [S1, unofficial]. Total vertices per subblock are limited to 128 (addressable with 8-bit indices), and total triangles are limited to 254 [S1, unofficial].

## Subtopics

### [Meshlet Extraction using Mesh Shader Pipelines](1874-meshlet-extraction-using-mesh-shader-pipelines.md)
A real-time isosurface extraction method that uses coupled task and mesh shaders to partition scalar volumes into size-constrained meshlets, addressing the lack of control over meshlet size constraints in its parent method, [Poisson Meshlet-Based (PMB) Extraction](1860-poisson-meshlet-based-pmb-extraction.md). It replaces PMB's single compute shader with a task/mesh shader pair to reduce thread-level divergence, enable hierarchical partitioning via task shader tree-expansion, and allow asynchronous execution for pipeline overlap.

### [Parallel Marching Blocks](3370-parallel-marching-blocks.md)
Parallel Marching Blocks (PMB) is a block-based isosurface extraction algorithm that generates indexed triangle meshes from 3D scalar volumes on the GPU. It solves the problem of redundant vertex data in traditional marching cubes by ensuring vertices are shared between simplices within a processing block, producing compact meshlets suitable for mesh shader pipelines. Unlike standard marching cubes which output a triangle soup, PMB avoids internal duplication within a block, though vertices on block boundaries are duplicated and must be welded by the consumer for a fully connected mesh.

## Operating characteristics
*   **Time Complexity:** Extraction performance is optimized when each thread analyzes $R=8$ cells and workgroup size is $T=32$ [S1, unofficial].
*   **Memory Complexity:** Memory interface size is limited by $C=128$ occupied cells per subblock [S1, unofficial].
*   **Measured Performance:** Outperforms the state-of-the-art Parallel Marching Blocks extraction approach by 12% to 27% for a variety of models and isovalue ranges [S1, unofficial]. The extracted meshlet representation results in a rendering time that is only 20% higher on average for large volumes than an offline-optimized meshlet representation, despite performing meshlet creation three orders of magnitude faster [S1, unofficial].
*   **Hardware:** Evaluated on NVIDIA RTX 4090 GPU and Intel Core i9-13900K CPU running at 3.00 GHz [S1, unofficial].
*   **Strengths:** Improves rendering efficiency by extracting from larger blocks, increasing primitives per meshlet [S1, unofficial]. Reduces thread-level divergence by focusing extraction on occupied cells [S1, unofficial].
*   **Limitations:** Performance degrades with workgroup sizes $>32$ threads [S1, unofficial]. Normal cone culling is unsuitable due to high overhead and arbitrary normal spread [S1, unofficial].
*   **When to Use:** Dynamic isosurface exploration scenarios requiring real-time extraction and rendering [S1, unofficial].
*   **When Not to Use:** When normal cone culling is required, or when hardware does not support mesh shader pipelines with efficient subgroup communication [S1, unofficial].

## Background
The method was proposed in "Real-Time Meshlet Extraction from Scalar Volumes" [S1, unofficial]. It builds on the Parallel Marching Blocks (PMB) technique [LCDW16] and addresses limitations in previous approaches like Nishidate and Fujishiro [NF24] by hierarchically assigning subblocks of varying size without producing meshlets that exceed size constraints [S1, unofficial]. The implementation was tested in modern OpenGL 4.6 using the `GL_NV_mesh_shader` extension and experimented with Vulkan [S1, unofficial].

## Sources
[S1] Real-Time Meshlet Extraction from Scalar Volumes, https://doi.org/10.2312/egpgv.20261001, fetched 2026-10-26 (unofficial)
