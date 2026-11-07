# Rasterizing Bounding Geometry for Empty Space Skipping

Also called: Bounding Geometry Rasterization, Raster-based Bounding Box Culling, Rasterizing Bounding Geometry

## Overview
A volume rendering optimization technique that rasterizes bounding geometry to compute start and end positions for ray marching, skipping empty space by discarding fragments that are not at the minimum depth value [S1, unofficial]. It exploits hardware-accelerated rasterization to skip over empty space by rasterizing bounding geometries, such as bounding boxes of leaf nodes of a tree structure [S2, unofficial]. Unlike standard octree empty space skipping, this method moves the major cost of empty space skipping into a rasterization stage that produces per-pixel lists of non-empty ray segments [S2, unofficial]. It is agnostic of the underlying data structure used for managing volume data, such as page-table hierarchies or octrees [S1, unofficial].

## How it works
**Inputs:**
*   Volume texture `gridTex` containing maximum iso values per voxel [S1, unofficial].
*   Threshold iso value for determining active voxels [S1, unofficial].
*   Coarse grid dimensions (e.g., $32 \times 32 \times 25$ for a $256 \times 256 \times 200$ volume with $8^3$ voxel cells) [S1, unofficial].

**Outputs:**
*   Ray start and end positions for each pixel [S1, unofficial].
*   Rendered volume image [S1, unofficial].

**Data Structures:**
*   `gridTex`: 3D texture where each voxel stores the maximum iso value of the underlying fine-grained volume region [S1, unofficial].
*   Bounding Geometry: A set of quads (or triangles) representing the boundaries between active and inactive regions in the coarse grid [S1, unofficial].

**Steps:**

1.  **Initialization:** Generate the volume texture `gridTex`. For each voxel in the coarse grid, compute the maximum iso value from the underlying fine-grained volume data [S1, unofficial].
2.  **Generate Bounding Geometry:** Given a threshold iso value, determine active voxels. A grid voxel is active if its value in `gridTex` is above the threshold [S1, unofficial]. For each pair of neighboring voxels in `gridTex`, generate a quad if one voxel is active and the other is inactive. This geometry represents the boundary of visible regions [S1, unofficial]. This process must be repeated whenever the iso value changes [S1, unofficial].
3.  **First Pass (Depth Buffering):** Render the bounding geometry to an offscreen buffer. Store the minimum and maximum depth values for each fragment covered by the geometry [S1, unofficial].
4.  **Second Pass (Rendering):** Render the front faces of the bounding geometry again to the main buffer.
5.  **Fragment Discard:** In the fragment shader, discard fragments that are not at the minimum depth value. This ensures that only the front-most entry point into a non-empty region is processed [S1, unofficial].
6.  **Ray Setup:** Compute the start and end positions for the ray using the minimum and maximum depth values obtained from the first pass [S1, unofficial]. The fragment positions of the front and back faces of the rasterized bounding geometry serve as these start and end points [S2, unofficial].
7.  **Ray Marching:** March the ray through the volume using the computed start and end positions [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
# gridTex is a 3D texture of size (W, H, D)
# Each voxel gridTex[i, j, k] stores the max iso value of the corresponding coarse cell
gridTex = compute_max_iso_texture(volume_data, coarse_grid_size)

# Generate Bounding Geometry
# active_mask[i, j, k] is true if gridTex[i, j, k] > threshold
active_mask = gridTex > threshold

bounding_geometry = []
for each voxel (i, j, k) in active_mask:
    if active_mask[i, j, k]:
        # Check neighbors to find boundaries
        # Generate quads for faces adjacent to inactive neighbors
        # This creates a mesh of bounding boxes around active regions
        pass

# First Pass: Render to offscreen buffer to get depth bounds
# Use a shader that outputs min_depth and max_depth for each fragment
render(bounding_geometry, target=offscreen_buffer, shader=depth_bounds_shader)

# Second Pass: Render to main buffer
render(bounding_geometry, target=main_buffer, shader=front_face_shader)

# Fragment Shader for Second Pass
def fragment_shader():
    # Get depth values from the first pass buffer
    min_depth = read_depth_from_offscreen_buffer()
    max_depth = read_depth_from_offscreen_buffer()
    
    # Discard if not at minimum depth
    if current_fragment_depth != min_depth:
        discard()
    
    # Compute ray start and end positions from min/max depth
    ray_start = depth_to_world_position(min_depth)
    ray_end = depth_to_world_position(max_depth)
    
    # March ray
    result = ray_march(ray_start, ray_end, volume_data)
    return result
```

**Parameters:**
*   Threshold iso value: Determines which voxels are considered active [S1, unofficial].
*   Coarse grid size: Determines the resolution of the bounding geometry [S1, unofficial].

**Edge Cases:**
*   Changing iso values requires regenerating the bounding geometry [S1, unofficial].
*   Precision issues in depth buffering may affect the accuracy of start/end positions [G].

## Subtopics

### [SparseLeap Rasterization](5154-sparseleap-rasterization.md)
SparseLeap is a method for efficient empty space skipping in very large, sparse volume data, such as terabyte-sized microscopy volumes, by exploiting hardware-accelerated rasterization to avoid hierarchy traversal on the GPU. It solves the problem of fragmented space subdivisions making empty regions hard to skip and the infeasibility of pre-computing well-adapted space subdivisions due to frequent changes in active objects. Unlike standard octree empty space skipping, SparseLeap scales better to large, sparse data by moving the major cost of empty space skipping into a rasterization stage that produces per-pixel lists of non-empty ray segments. It is agnostic of the underlying data structure used for managing volume data, such as page-table hierarchies or octrees.

## Operating characteristics
*   **Time Complexity:** Depends on the number of active bounding geometry fragments. Rasterization is hardware-accelerated, potentially faster than CPU/GPU tree traversal [S2, unofficial].
*   **Memory Complexity:** Requires storage for `gridTex` and the bounding geometry [S1, unofficial].
*   **Performance:** Can lead to fewer look-ups into the empty-space skipping data structure during rendering when compared to octree structures [S2, unofficial].
*   **Strengths:** Avoids hierarchy traversal on the GPU [S2, unofficial]. Agnostic of underlying data structure [S1, unofficial].
*   **Limitations:** Requires regenerating geometry when iso values change [S1, unofficial].

## Background
The technique was proposed by Liu et al. [LCDP13] and further utilized by Hadwiger et al. [HAAB+ 18] to avoid hierarchy traversal on the GPU [S2, unofficial]. It is part of the broader field of object-order empty space skipping [S2, unofficial].

## Sources
[S1] implement volume rendering with object order empty space skipping, https://stackoverflow.com/questions/12216008/implement-volume-rendering-with-object-order-empty-space-skipping, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
