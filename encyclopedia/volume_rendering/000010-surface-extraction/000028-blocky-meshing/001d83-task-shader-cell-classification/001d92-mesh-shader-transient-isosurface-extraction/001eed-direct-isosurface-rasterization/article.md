# Direct Isosurface Rasterization

Also called: Direct Isosurface Rasterization via Task and Mesh Shaders, Direct isosurface visualization, Transient isosurface extraction

## Overview
Direct Isosurface Rasterization is a rasterization-based approach for rendering isosurfaces that avoids intermediate explicit mesh storage by leveraging task and mesh shaders to identify subvolumes containing isosurface geometry and extract primitives on the fly for direct consumption by the rasterizer [S1]. It solves the problem of high memory bandwidth and storage requirements associated with indirect methods (like standard Marching Cubes) and the resolution-dependent performance costs of ray-marching [S1]. Unlike its siblings, it generates transient indexed triangles directly from volume data without precomputing or storing a persistent mesh structure [S1].

## How it works
The method operates in two main stages: occlusion culling to determine the Potentially Visible Set (PVS) of blocks, and transient extraction to render the isosurface [S1]. Both stages utilize the task and mesh shader pipeline [S1].

### Inputs and Outputs
*   **Input:** A scalar volume dataset $V$, a precomputed min-max grid storing the minimum and maximum data values for each block, and a target isovalue [S1].
*   **Output:** Rasterized pixels representing the isosurface at the specified isovalue [S1].

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

1.  **Occlusion Culling (Determining PVS):**
    *   Each task shader workgroup processes one block [S1].
    *   It performs a single texture lookup on the min-max grid for the block [S1].
    *   If the isovalue lies within $[min, max]$, the block is considered potentially occupied [S1].
    *   The task shader emits mesh shader workgroups for these potentially occupied blocks [S1].
    *   Mesh shader instances generate proxy geometry (e.g., bounding boxes) for the blocks identified by the task shader [S1].
    *   This proxy geometry is rasterized to perform conservative occlusion culling [S1].
    *   The result determines the final Potentially Visible Set (PVS) of blocks for the current frame [S1].

2.  **Transient Isosurface Extraction and Rendering:**
    *   Each task shader workgroup processes one block from the PVS [S1].
    *   It iterates through all $U \times V \times W$ cells in the block [S1].
    *   For each cell, it applies the Marching Cubes cell classification to determine if the isosurface intersects the cell [S1].
    *   Cells that produce geometry are marked as "occupied" [S1].
    *   The task shader creates a compact list of these occupied cells [S1].
    *   It emits mesh shader workgroups, assigning them subsets of the occupied cells to process [S1].
    *   Each mesh shader instance receives a list of occupied cells [S1].
    *   For each occupied cell, it performs the Marching Cubes algorithm to extract the specific triangle indices [S1].
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
CELLS_PER_BLOCK = U * V * W  # 256

# Standard Marching Cubes Edge and Triangle Tables (256 cases)
# Defined per standard implementation [G]
# EDGE_TABLE = [...]
# TRIANGLE_TABLE = [...]

def task_shader_process_block(block_idx, min_max_grid, isovalue):
    min_val, max_val = min_max_grid.lookup(block_idx)
    if isovalue < min_val or isovalue > max_val:
        return []  # Block empty, skip
    occupied_cells = []
    for cell_idx in range(CELLS_PER_BLOCK):
        if marching_cubes_classify(cell_idx, isovalue):
            occupied_cells.append(cell_idx)
    return occupied_cells

def mesh_shader_process_cells(occupied_cells, volume_data, isovalue):
    primitives = []
    for cell_idx in occupied_cells:
        # Extract triangles using standard Marching Cubes lookup [G]
        triangles = extract_triangles(cell_idx, volume_data, isovalue)
        primitives.extend(triangles)
    return primitives

def rasterize_primitives(primitives):
    # Pass primitives directly to rasterization hardware
    pass
```

### Parameters and Edge Cases
*   **Block Dimensions:** $U=8, V=8, W=4$ is a sensible setting that allows 8-bit indexing of the 256 cells contained in a block [S1].
*   **Visibility State:** The pipeline works on mutually exclusive sets of blocks depending on their visibility state in the current and previous frame [S1].
*   **Culling Conservatism:** Occlusion culling is performed conservatively rather than probabilistically to ensure no visible geometry is incorrectly culled [S1].
*   **Hardware Adaptation:** While described using NVIDIA terminology (warps, task/mesh shaders), the method applies to other hardware by substituting equivalent constructs (e.g., AMD wavefronts) [S1].

## Operating characteristics
*   **Performance:** At 3840x2160 resolution, the method achieved average draw times of 0.19ms (CT Head), 0.26ms (Vertebra), 1.26ms (Chameleon), and 0.29ms (Timeseries Supernova), compared to 1.54ms, 1.95ms, 2.59ms, and 1.41ms for min-max octree ray marching respectively [S1].
*   **Scaling:** Performance is less dependent on display resolution than ray-marching because rasterization units are highly optimized, whereas ray-marching performance has a linear dependence on resolution [S1]. Hypothesized to lead to better performance than ray marching approaches as rendering resolution increases [S1].
*   **Strengths:** Requires little preprocessing and negligible additional memory because the mesh is transient [S1]. Affords straightforward integration into existing rendering pipelines and supports modern features like multi-view stereo [S1]. Particularly suitable for applications where isosurface geometry is highly variable and requires frequent isovalue adjustment, such as interactive analysis scenarios [S1].
*   **Limitations:** Requires support for task and mesh shader pipelines. Performance depends on the efficiency of the underlying rasterization hardware [S1].

## Background
*   **Origin:** First published on 20 March 2023 [S1].
*   **Development:** The approach builds on raster occlusion culling techniques [KT14, LJSL21] and the Marching Cubes algorithm [LC87]. It was designed to address the memory bandwidth and resolution-scaling bottlenecks of traditional direct and indirect isosurface visualization methods [S1].
*   **Usage:** Used in real-time volume rendering pipelines where interactive isovalue adjustment and high-resolution rendering are required [S1].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
