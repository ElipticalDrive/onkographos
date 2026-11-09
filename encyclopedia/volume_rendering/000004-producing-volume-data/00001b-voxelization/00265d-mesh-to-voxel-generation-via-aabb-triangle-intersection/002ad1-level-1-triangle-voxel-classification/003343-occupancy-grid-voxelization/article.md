# Occupancy Grid Voxelization

Also called: Discretize, Scene Discretization to Occupancy Grid

## Overview
Occupancy Grid Voxelization converts a target mesh shape into a binary occupancy grid aligned with the sensor resolution for use as an optimization reference [S1, unofficial]. It computes this grid by checking if each voxel box intersects with any mesh triangle [S2, unofficial].

## How it works
**Inputs:**
*   `vertices`: List of 3D vertex coordinates ($V \in \mathbb{R}^{N_v \times 3}$) [S2, unofficial].
*   `faces`: List of triangle indices ($F \in \mathbb{Z}^{N_f \times 3}$) [S2, unofficial].
*   `resolution`: Tuple $(H, W, D)$ defining the grid dimensions [S1, unofficial].
*   `bounds`: Sensor bounding box extents used to compute voxel size [S1, unofficial].

**Outputs:**
*   `occ`: A 3D tensor of shape $(H, W, D)$ storing binary occupancy values (0 or 1) or face indices. In C++ implementations, this is `Eigen::Tensor<int, 3, Eigen::RowMajor>` [S2, unofficial].

**Data Structures:**
*   **Coordinate Grids**: Generated using `dr.meshgrid` with 'ij' indexing to map voxel indices to physical coordinates [S1, unofficial].

**Steps:**
1.  **Compute Voxel Size**: Calculate voxel dimensions by dividing the sensor bounding box extents by the resolution $(H, W, D)$ [S1, unofficial].
2.  **Initialize Grid**: Create an occupancy grid of size $H \times W \times D$ [S2, unofficial].
3.  **Iterate Voxels**: Loop over all voxel linear indices $i$ from $0$ to $H \times W \times D - 1$ [S2, unofficial].
    *   Decode linear index $i$ to 3D coordinates $(h, w, d)$:
        $$ d = i \pmod D $$
        $$ w = \lfloor i / D \rfloor \pmod W $$
        $$ h = \lfloor i / (W \times D) \rfloor $$
    *   Define voxel bounds in index space:
        $$ \text{min\_idx} = (w, h, d) $$
        $$ \text{max\_idx} = (w + 1, h + 1, d + 1) $$
4.  **Check Intersection**: For each voxel, iterate over all faces $f$ from $0$ to $N_f - 1$ [S2, unofficial].
    *   Retrieve vertices $v_1, v_2, v_3$ for face $f$ [S2, unofficial].
    *   Perform triangle-box intersection test:
        $$ \text{overlap} = \text{triangle\_box\_intersection}(\text{min\_idx}, \text{max\_idx}, v_1, v_2, v_3) $$
    *   If `overlap` is true:
        *   Set `occ(h, w, d) = 1` (for binary) or store face index $f$ (for color-preserving).
        *   Break the face loop for this voxel [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def occupancy_grid_voxelization(vertices, faces, resolution, bounds):
    H, W, D = resolution
    min_x, min_y, min_z, max_x, max_y, max_z = bounds
    dx = (max_x - min_x) / W
    dy = (max_y - min_y) / H
    dz = (max_z - min_z) / D
    
    occ = [[[0] * D for _ in range(W)] for _ in range(H)]
    
    for i in range(H * W * D):
        d = i % D
        w = (i // D) % W
        h = (i // (W * D)) % H
        
        voxel_min = [min_x + w * dx, min_y + h * dy, min_z + d * dz]
        voxel_max = [min_x + (w + 1) * dx, min_y + (h + 1) * dy, min_z + (d + 1) * dz]
        
        for f_idx in range(len(faces)):
            v1 = vertices[faces[f_idx][0]]
            v2 = vertices[faces[f_idx][1]]
            v3 = vertices[faces[f_idx][2]]
            
            if triangle_box_intersection(voxel_min, voxel_max, v1, v2, v3):
                occ[h][w][d] = 1
                break
                    
    return occ

def triangle_box_intersection(box_min, box_max, v1, v2, v3):
    """Standard triangle-AABB intersection test [G]."""
    # Slab method: check overlap along each axis
    for axis in range(3):
        p1, p2, p3 = v1[axis], v2[axis], v3[axis]
        min_p = min(p1, p2, p3)
        max_p = max(p1, p2, p3)
        if max_p < box_min[axis] or min_p > box_max[axis]:
            return False
    return True
```

**Parameters:**
*   `resolution`: Not given in the sources. [S1, unofficial]
*   `bounds`: Defined by the sensor bounding box extents [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision**: Floating-point errors in intersection tests may cause missed voxels or false positives [G].
*   **Degenerate Input**: Triangles with zero area or vertices outside the bounding box should be handled or filtered [S2, unofficial].

## Subtopics

### Color-Preserving Occupancy Voxelization
Computes an occupancy grid where voxel values store the index of the first face that intersects the voxel, preserving color information by mapping face indices to voxel values [S2, unofficial]. It follows the same iteration and intersection logic as the binary variant but stores `f` (or `num_faces()` if `f==0`) instead of `1` [S2, unofficial]. **Pitfall**: Face index 0 is treated as 'not voxelized' space by default. If face 0 overlaps, store `num_faces()` in the voxel instead of 0 to distinguish it from empty space [S2, unofficial].

### Tetravoxelization Grid Population
Populates a regular voxel grid by tetravoxelizing input 3D models and mapping binary occupancy results to grid values [S3, unofficial]. It initializes a tetravoxelizer with grid subdivisions, processes each model component with geometry, topology, and AABB, and computes voxelization into a 1D array indexed as `y * X * Z + z * X + x` [S3, unofficial]. It then iterates over grid dimensions $(y, x, z)$, calculates `positionIndex = y * _numDivs.x * _numDivs.z + z * _numDivs.x + x`, and if `_voxelOpenGL[positionIndex] == 1`, sets `_grid[positionIndex]._value = VOXEL_FREE` and marks `activeVoxels = true` [S3, unofficial]. **Pitfall**: If tetravoxelization yields no active voxels (empty model or misaligned AABB), the method falls back to `fillNaive(model)` to prevent empty grids [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(H \times W \times D \times N_f)$, where $H, W, D$ are grid dimensions and $N_f$ is the number of faces [S2, unofficial]. This is a brute-force approach [G].
*   **Memory Complexity**: $O(H \times W \times D)$ for the occupancy grid [G].
*   **Performance**: Performance depends on the efficiency of the triangle-box intersection test. Parallelization over voxels is straightforward [S2, unofficial].
*   **Strengths**: Simple to implement; provides a dense, aligned grid suitable for optimization references [S1, unofficial].
*   **Limitations**: Computationally expensive for large scenes or high resolutions [G].
*   **When to use**: When a binary or face-indexed occupancy grid is required for optimization or visualization, and the mesh complexity is manageable [G].
*   **When not to use**: For real-time applications or very large meshes where sparse representations or hierarchical structures are more appropriate [G].

## Background
Originating from inverse rendering for tomographic volumetric additive manufacturing, this method discretizes target shapes into occupancy grids for optimization [S1, unofficial]. It is also used in general mesh processing for converting polygonal models into voxel-based representations [S2, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S2] Mesh Voxelization - GitHub, https://github.com/davidstutz/mesh-voxelization, fetched 2026-10-26 (unofficial)
[S3] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
