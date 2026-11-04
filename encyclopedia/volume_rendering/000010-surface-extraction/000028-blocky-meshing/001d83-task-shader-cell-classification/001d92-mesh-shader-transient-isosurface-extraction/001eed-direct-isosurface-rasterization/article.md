# Direct Isosurface Rasterization

Also called: Direct Isosurface Rasterization via Task and Mesh Shaders, Direct isosurface visualization, Transient isosurface extraction

## Overview
Direct Isosurface Rasterization is a rasterization-based approach for rendering isosurfaces that avoids intermediate explicit mesh storage by leveraging task and mesh shaders to identify subvolumes containing isosurface geometry and extract primitives on the fly for direct consumption by the rasterizer [S1]. It solves the problem of high memory bandwidth and storage requirements associated with indirect methods (like standard Marching Cubes) and the resolution-dependent performance costs of ray-marching [S1]. Unlike its siblings, it generates transient indexed triangles directly from volume data without precomputing or storing a persistent mesh structure [S1].

## How it works
The method operates in two main stages: occlusion culling to determine the Potentially Visible Set (PVS) of blocks, and transient extraction to render the isosurface [S1]. Both stages utilize the task and mesh shader pipeline [S1].

### Inputs and Outputs
*   **Input:** A scalar volume dataset $V$. Precomputed min-max grid storing the minimum and maximum data values for each block [S1].
*   **Output:** Rasterized pixels representing the isosurface at a specified isovalue [S1].

### Data Structures

**Block Definition**
A block is a virtual subvolume comprised of $U \times V \times W$ cells [S1].
*   Dimensions: $U=8, V=8, W=4$ [S1].
*   Total cells per block: $256$ [S1].
*   Indexing: Cells within a block are indexed using 8-bit integers (0 to 255) [S1].
*   Note: Blocks are a virtual subdivision; the underlying volume data does not require rearrangement [S1].

**Min-Max Grid**
*   Format: 2-channel volume texture [S1].
*   Resolution: Matches the block grid resolution of the volume [S1].
*   Content: Stores $(min, max)$ values for each block [S1].
*   Purpose: Allows occupancy determination with a single texture lookup [S1].

**Memory Interface Block**
*   Purpose: Shared memory interface between a task shader workgroup and its emitted mesh shader workgroups [S1].
*   Content: Provides mesh shader workgroups with a compact list of occupied cells to process [S1].
*   Benefit: Increases the chance of high warp occupancy by grouping related cell processing [S1].

### Algorithm Steps

The pipeline processes mutually exclusive sets of blocks depending on their visibility state in the current and previous frame [S1].

#### 1. Occlusion Culling (Determining PVS)
This stage identifies which blocks contain visible isosurface geometry [S1].

1.  **Task Shader (Block Classification):**
    *   Each task shader workgroup processes one block [S1].
    *   It performs a texture lookup on the min-max grid for the block [S1].
    *   If the isovalue lies within $[min, max]$, the block is considered potentially occupied [S1].
    *   The task shader emits mesh shader workgroups for these potentially occupied blocks [S1].

2.  **Mesh Shader (Proxy Generation):**
    *   Mesh shader instances generate proxy geometry (e.g., bounding boxes or simplified representations) for the blocks identified by the task shader [S1].
    *   This proxy geometry is rasterized to perform conservative occlusion culling [S1].
    *   The result determines the final Potentially Visible Set (PVS) of blocks for the current frame [S1].

#### 2. Transient Isosurface Extraction and Rendering
This stage extracts and renders the actual isosurface geometry for the PVS blocks [S1].

1.  **Task Shader (Cell Classification):**
    *   Each task shader workgroup processes one block from the PVS [S1].
    *   It iterates through all $U \times V \times W$ cells in the block [S1].
    *   For each cell, it applies the **Marching Cubes cell classification** [LC87] to determine if the isosurface intersects the cell [S1].
    *   Cells that produce geometry are marked as "occupied" [S1].
    *   The task shader creates a compact list of these occupied cells [S1].
    *   It emits mesh shader workgroups, assigning them subsets of the occupied cells to process [S1].

2.  **Mesh Shader (Primitive Extraction):**
    *   Each mesh shader instance receives a list of occupied cells [S1].
    *   For each occupied cell, it performs the **Marching Cubes algorithm** to extract the specific triangle indices [S1].
    *   It generates indexed triangles representing the isosurface segment for that cell [S1].
    *   The extracted primitives are passed directly to the rasterization units [S1].

3.  **Rasterization:**
    *   The rasterizer consumes the primitives directly [S1].
    *   No intermediate mesh storage is used [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
U, V, W = 8, 8, 4
CELLS_PER_BLOCK = U * V * W # 256

# Shared Memory Structure for Task -> Mesh communication
class MemoryInterfaceBlock:
    occupied_cell_indices: list[int] # List of 8-bit indices (0-255)
    count: int

# Task Shader Workgroup Function
def task_shader_workgroup(block_id: int):
    # 1. Check Block Occupancy (for Culling or Extraction)
    min_val, max_val = load_min_max(block_id) # From min-max grid texture
    
    if not isovalue_in_range(min_val, max_val):
        return # Block is empty or fully outside isovalue

    # 2. Identify Occupied Cells within the Block
    occupied_cells = []
    for cell_idx in range(CELLS_PER_BLOCK):
        # Get voxel values for the 8 vertices of the cell
        # Note: cell_idx maps to specific (x,y,z) coordinates in the block
        vertex_values = get_cell_vertices(block_id, cell_idx)
        
        # Apply Marching Cubes classification
        # Returns true if the isosurface intersects this cell
        if marching_cubes_classify(vertex_values, isovalue):
            occupied_cells.append(cell_idx)

    if len(occupied_cells) == 0:
        return

    # 3. Emit Mesh Shader Workgroups
    # Distribute occupied cells to mesh shader workgroups
    # The exact distribution strategy (e.g., round-robin or chunking) 
    # is implementation-specific but aims to balance load [S1].
    
    interface = MemoryInterfaceBlock()
    interface.occupied_cell_indices = occupied_cells
    interface.count = len(occupied_cells)
    
    # Emit mesh shader workgroups, passing 'interface' as shared memory
    emit_mesh_shader_workgroups(interface)

# Mesh Shader Workgroup Function
def mesh_shader_workgroup(interface: MemoryInterfaceBlock):
    # 1. Process Assigned Cells
    for cell_idx in interface.occupied_cell_indices:
        # Get voxel values for the 8 vertices of the cell
        vertex_values = get_cell_vertices(interface.block_id, cell_idx)
        
        # 2. Extract Geometry using Marching Cubes
        # Returns a list of triangle indices (vertex indices in the output buffer)
        triangles = marching_cubes_extract(vertex_values, isovalue)
        
        # 3. Output Triangles
        for tri in triangles:
            write_triangle_to_output_buffer(tri)
```

### Parameters
*   **Block Dimensions:** $U=8, V=8, W=4$ [S1].
*   **Isovalue:** User-defined scalar threshold [S1].
*   **Typical Use Case:** Interactive analysis scenarios requiring frequent isovalue adjustment [S1].

### Edge Cases and Pitfalls
*   **Visibility State:** The pipeline works on mutually exclusive sets of blocks depending on their visibility state in the current and previous frame [S1].
*   **Conservative Culling:** The occlusion culling must be conservative rather than probabilistic to avoid missing visible geometry [S1].
*   **Precision:** Standard Marching Cubes precision applies; no special handling is noted for floating-point precision issues in the sources [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in Big-O notation, but performance is competitive with ray-marching and significantly better at high resolutions [S1].
*   **Memory Complexity:** Negligible additional memory compared to indirect methods, as no explicit mesh is stored [S1]. Requires storage for the min-max grid [S1].
*   **Measured Performance:** At 3840x2160 resolution, average draw times were:
    *   CT Head: 0.19ms (vs 1.54ms for min-max octree ray marching) [S1].
    *   Vertebra: 0.26ms (vs 1.95ms) [S1].
    *   Chameleon: 1.26ms (vs 2.59ms) [S1].
    *   Timeseries Supernova: 0.29ms (vs 1.41ms) [S1] [S1].
*   **Scaling:** Performance is less dependent on display resolution than ray-marching because rasterization units are highly optimized, whereas ray-marching performance has a linear dependence on resolution [S1].
*   **Strengths:**
    *   Low memory bandwidth and storage overhead [S1].
    *   High performance at high resolutions [S1].
    *   Suitable for interactive analysis with frequent isovalue changes [S1].
    *   Straightforward integration into existing rasterization pipelines [S1].
*   **Limitations:**
    *   Requires preprocessing to compute min-max grids [S1].
    *   Relies on modern graphics hardware features (task/mesh shaders) [S1].
*   **When to Use:** Applications where isosurface geometry is highly variable, requiring frequent isovalue adjustment, and high resolution is desired [S1].
*   **When Not to Use:** Legacy hardware without task/mesh shader support, or scenes where preprocessing min-max grids is prohibitive [S1].

## Background
*   **Origin:** First published on 20 March 2023 [S1].
*   **Paper:** "Efficient Direct Isosurface Rasterization of Scalar Volumes" [S1].
*   **History:** Builds on the concept of direct isosurface visualization (ray marching) and indirect visualization (Marching Cubes) [S1]. It leverages modern graphics hardware features like task and mesh shading pipelines, which were introduced to address the inefficiencies of traditional geometry shaders [S1]. The method adapts techniques from raster occlusion culling [KT14, LJSL21] to volume rendering [S1].
*   **Usage:** Suitable for interactive analysis of static and dynamic data sets, including time-varying data streams [S1]. It supports modern graphics features like multi-view stereo [S1].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
