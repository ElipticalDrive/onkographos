# GPU-accelerated Marching Cubes Pipeline

Also called: Six-stage GPU-accelerated Marching Cubes methodology

## Overview
This method is a six-stage hybrid CPU-GPU pipeline for isosurface extraction that maximizes performance by balancing memory hierarchy usage and parallelizing the Marching Cubes algorithm [S1]. It differs from sibling methods like [Capacity-Safe Marching Cubes Pipeline](4596-capacity-safe-marching-cubes-pipeline.md) by offloading the spatial traversal and active cell identification to the CPU using pre-constructed spatial data structures, while the GPU handles the exact buffer allocation and triangle generation in parallel [S1]. Unlike [GPU Marching Cubes with Scan-Based Compaction](6632-gpu-marching-cubes-with-scan-based-compaction.md), which typically handles all active cells on the GPU, this approach reduces GPU memory pressure and bandwidth by transferring only the necessary active cell indices from CPU to GPU [S1].

## How it works
**Inputs:**
*   Volume dataset $V$ of dimensions $N_x \times N_y \times N_z$ [S1].
*   Isovalue $h$ [S1].
*   Spatial data structure type (k-d tree, interval tree, quadtree, or octree) [S1].

**Outputs:**
*   Vertex buffer $P$ containing 3D positions of isosurface vertices [S1].
*   Normal buffer $N$ containing surface normals [S1].
*   Rendered isosurface [S1].

**Data Structures:**
*   **Volume Data:** Allocated in RAM during Stage 1, then freed from RAM in Stage 2 while remaining in VRAM [S1].
*   **Spatial Structure:** Stored in Main Memory (RAM) for CPU-side traversal [S1].
*   **Active Cell List:** A list of indices corresponding to voxels intersecting the isosurface, transferred from CPU to GPU [S1].
*   **Vertex Buffers ($P, N$):** Allocated in VRAM with exact sizes determined in Stage 4 [S1].
*   **Lookup Tables:** Stored in Texture Memory for fast read-only access by GPU kernels [S1].

**Parameters:**
*   **Block Size:** The number of threads per block, chosen to balance workload: not too low (to avoid underutilization) and not too high (to avoid thread launch/termination overhead) [S1].
*   **Memory Types:**
    *   *Texture Memory:* Used for lookup tables and volume data [S1].
    *   *Shared Memory:* Used for computing edge intersections within a thread block [S1].

**Steps:**
1.  **CPU: Data Loading and Allocation.** Read volume dataset $V$ ($N_x \times N_y \times N_z$). Allocate data in main memory (RAM) and video memory (VRAM). Allocate extra memory for span-space related to volume data (except for octree) [S1].
2.  **CPU: Spatial Structure Construction.** Construct one of the spatial data structures from volume data. Store structure only in main memory. Free volume data from main memory (keep in VRAM) [S1].
3.  **CPU: Active Cell Identification.** Given isovalue $h$, search in the data structure to traverse only nodes corresponding to active volume cells. Create a list of these active cells. Transfer list to GPU via communication bus [S1].
4.  **GPU: Buffer Allocation.** Classify each active cell into one of the 15 marching cube cases by comparing $h$ to eight cell vertices. Create cell index. Determine total number of vertices needed to render isosurface. Allocate exact size of video memory for two vertex buffers: one for vertices, one for normals [S1].
5.  **GPU: Triangle Generation.** Traverse list of active cells. For each cell, calculate isosurface intersections with 12 edges by interpolating vertices and normals from volume data. Use cell index and intersections to obtain list of vertices and normals. Write them to respective vertex buffers [S1].
6.  **GPU: Rendering.** Render volume data from the vertex buffers [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard operations defined inline for self-containment
def classify_cell(vertex_values, isovalue):
    # Returns integer 0-255 based on sign of (v - h) for 8 vertices
    index = 0
    for i, v in enumerate(vertex_values):
        if v < isovalue:
            index |= (1 << i)
    return index

def count_vertices_for_case(cell_index):
    # Returns vertex count for the 15 non-degenerate MC cases
    # Case 0, 15: 0 vertices. Others: 2, 4, 6, or 8 depending on topology.
    # Simplified mapping for standard MC:
    counts = {0: 0, 15: 0, 1: 2, 2: 2, 3: 4, 4: 2, 5: 4, 6: 4, 7: 6,
              8: 4, 9: 4, 10: 6, 11: 6, 12: 4, 13: 4, 14: 6}
    return counts.get(cell_index, 0)

def interpolate_edge(v1, v2, h):
    # Linear interpolation t = (h - v1) / (v2 - v1)
    denom = v2 - v1
    if abs(denom) < 1e-9:
        return (v1 + v2) * 0.5
    return v1 + (h - v1) / denom

def get_edge_intersections(cell_index, vertex_values, isovalue):
    # Returns list of (position, normal) for the 12 edges
    intersections = []
    edge_pairs = [(0,1),(1,2),(2,3),(3,0), (4,5),(5,6),(6,7),(7,4),
                  (0,4),(1,5),(2,6),(3,7)]
    for e1, e2 in edge_pairs:
        v1, v2 = vertex_values[e1], vertex_values[e2]
        if (v1 < isovalue) != (v2 < isovalue):
            t = interpolate_edge(v1, v2, isovalue)
            pos = (v1 + v2) * 0.5 # Approximate position along edge
            normal = (v1 + v2) * 0.5 # Approximate normal from gradient
            intersections.append((pos, normal))
    return intersections

# Stage 1 & 2: CPU Preprocessing
def cpu_preprocessing(volume_data, dimensions, structure_type):
    ram_volume = volume_data.copy() # Allocate in RAM
    vram_volume = volume_data.copy() # Allocate in VRAM
    spatial_structure = build_spatial_tree(ram_volume, structure_type) # [G]
    del ram_volume # Free RAM
    return spatial_structure, vram_volume

# Stage 3: CPU Active Cell Search
def cpu_find_active_cells(spatial_structure, isovalue):
    active_cells = []
    for node in spatial_structure.traverse(isovalue):
        active_cells.append(node.index)
    return active_cells

# Stage 4: GPU Buffer Allocation (First Pass)
def gpu_allocate_buffers(active_cells, vram_volume, isovalue):
    total_vertices = 0
    for cell_index in active_cells:
        vertex_values = [vram_volume[cell_index][i] for i in range(8)]
        mc_index = classify_cell(vertex_values, isovalue)
        total_vertices += count_vertices_for_case(mc_index)
    
    buffer_P = allocate_vram(total_vertices * 3) # 3 floats per vertex
    buffer_N = allocate_vram(total_vertices * 3) # 3 floats per normal
    return buffer_P, buffer_N

# Stage 5: GPU Triangle Generation (Second Pass)
def gpu_generate_triangles(active_cells, vram_volume, buffer_P, buffer_N, isovalue):
    for cell_index in active_cells:
        vertex_values = [vram_volume[cell_index][i] for i in range(8)]
        mc_index = classify_cell(vertex_values, isovalue) # Recomputed as per source
        
        intersections = get_edge_intersections(mc_index, vertex_values, isovalue)
        
        # Write to buffers using shared memory for intra-block sharing [S1]
        for pos, normal in intersections:
            write_to_buffer(buffer_P, pos)
            write_to_buffer(buffer_N, normal)

# Stage 6: Rendering
def gpu_render(buffer_P, buffer_N):
    draw_geometry(buffer_P, buffer_N) # [G]
```

**Edge Cases and Pitfalls:**
*   **Block Size Selection:** Choosing a block size that is too low underutilizes threads, while a size that is too high causes overhead in thread management [S1].
*   **Redundant Calculation:** The cell index calculation is performed in both Stage 4 and Stage 5. This is intentional as it is faster on the GPU to recompute than to store and retrieve from an array [S1].
*   **Memory Hierarchy:** Failure to use Texture Memory for read-only data (lookup tables, volume data) and Shared Memory for intra-block data sharing (edge intersections) will result in suboptimal performance [S1].

## Subtopics

### [Marching Cubes with Spatial Acceleration Structures](14738-marching-cubes-with-spatial-acceleration-structures.md)
A variant that accelerates the search for active cells using spatial data structures (k-d tree, interval tree, quadtree, octree) implemented on the GPU, but where the search itself is performed on the CPU.

## Operating characteristics
*   **Complexity:** The CPU handles the spatial traversal ($O(N)$ in worst case, but optimized by structure) and the GPU handles the parallel classification and interpolation ($O(A)$ where $A$ is the number of active cells) [S1].
*   **Performance:** The method balances CPU and GPU tasks to maximize performance [S1]. It relies on the parallelization of the active cell list processing on the GPU [S1].
*   **Memory:** Requires allocation for volume data in VRAM, spatial structure in RAM, and exact-sized vertex buffers in VRAM. Extra memory for span-space is needed for non-octree structures [S1].
*   **Strengths:** Avoids buffer overflow by exact allocation; leverages CPU for spatial traversal which can be more efficient for sparse data; uses GPU memory hierarchy (Texture/Shared) for speed [S1].
*   **Limitations:** Requires data transfer between CPU and GPU for the active cell list; dependent on the efficiency of the spatial structure construction and traversal [S1].

## Background
The methodology was proposed to accelerate the Marching Cubes technique for volumetric visualization by properly utilizing the GPU memory hierarchy and parallel processing capabilities [S1]. It builds upon earlier proposals by Buatois et al., Johansson, Martin et al., Pascucci, Reck et al., and Tatarchuk et al. [S1]. The specific implementation details, including the six-stage scheme and CUDA kernel optimizations, are described in the source [S1].

## Sources
[S1] Marching cubes technique for volumetric visualization accelerated with graphics processing units, https://doi.org/10.1007/s13173-012-0097-z, fetched 2026-10-26
