# Occupancy Grid Voxelization

Also called: Discretize, Grid Occupancy Voxelization, Scene Discretization to Occupancy Grid, Vertex-based Voxelization

## Overview
Occupancy Grid Voxelization converts a target mesh shape into a binary occupancy grid aligned with a sensor resolution, serving as an optimization reference or initial volume representation [S2, unofficial]. It builds on [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md) by explicitly mapping vertex positions to discrete grid indices and rendering occupied cells as unit cubes, or by iterating over all voxels to test for triangle intersection [S1, unofficial][S3, unofficial]. Unlike [Level 2 Triangle-Voxel Classification](5512-level-2-triangle-voxel-classification.md), which refines the classification, this method establishes the coarse binary presence of geometry [S3, unofficial].

## How it works
The method operates in two primary modes: vertex-based discretization and exhaustive voxel-face intersection. Both require defining a 3D grid aligned with the scene's bounding box.

### Inputs and Outputs
**Inputs:**
*   **Mesh:** A set of vertices $V = \{v_0, \dots, v_{N_v-1}\}$ where $v_i \in \mathbb{R}^3$, and faces $F = \{f_0, \dots, f_{N_f-1}\}$ where $f_j = (v_a, v_b, v_c)$ are indices into $V$ [S3, unofficial].
*   **Resolution:** A scalar or vector defining the grid dimensions. In the vertex-based approach, this is often a fixed `SAMPLE_RATE` [S1, unofficial]. In the intersection approach, it is derived from the sensor bounding box extents divided by the voxel size [S2, unofficial].

**Outputs:**
*   **Binary Occupancy Grid:** A 3D array `occ` where `occ(h, w, d) = 1` if the voxel is occupied, `0` otherwise. The layout is typically row-major or flattened 1D indexed by $y \cdot X \cdot Z + z \cdot X + x$ [S3, unofficial][S4, unofficial].

### Data Structures
1.  **Voxel Grid:**
    *   **Vertex-based:** `bool voxelData[x][y][z]` allocated as a 3D array of dimensions $(SAMPLE\_RATE + 1) \times (SAMPLE\_RATE + 1) \times (SAMPLE\_RATE + 1)$ [S1, unofficial]. Memory is allocated sequentially: `voxelData[x] -> voxelData[x][y] -> voxelData[x][y][z]` [S1, unofficial].
    *   **Intersection-based:** `Eigen::Tensor<int, 3, Eigen::RowMajor> occ` or a 1D array `_voxelOpenGL` [S3, unofficial][S4, unofficial].

2.  **Bounding Box:**
    *   `minX, maxX, minY, maxY, minZ, maxZ` initialized to $\pm \infty$ (or $\pm 99999999$) and updated by iterating over all mesh vertices [S1, unofficial].

### Algorithm: Vertex-Based Discretization
This approach maps vertices directly to grid indices, assuming the mesh fills the space between vertices or using the vertices as anchors for unit cubes.

1.  **Initialize Bounds:** Set `minX = maxX = v_0.x`, etc., for all axes. Iterate over all vertices $v \in V$ to update bounds:
    $$
    \begin{aligned}
    minX &= \min(minX, v.x) \\
    maxX &= \max(maxX, v.x)
    \end{aligned}
    $$
    (Repeat for $y, z$) [S1, unofficial].

2.  **Compute Index Spans:** Calculate the number of voxels along each axis:
    $$
    \begin{aligned}
    X_{span} &= \frac{|maxX - minX|}{SAMPLE\_RATE} \\
    Y_{span} &= \frac{|maxY - minY|}{SAMPLE\_RATE} \\
    Z_{span} &= \frac{|maxZ - minZ|}{SAMPLE\_RATE}
    \end{aligned}
    $$
    [S1, unofficial].

3.  **Map Vertices to Grid:** For each vertex $v$, compute grid indices:
    $$
    \begin{aligned}
    x &= \frac{v.x + |minX|}{X_{span}} \\
    y &= \frac{v.y + |minY|}{Y_{span}} \\
    z &= \frac{v.z + |minZ|}{Z_{span}}
    \end{aligned}
    $$
    Set `voxelData[x][y][z] = true` [S1, unofficial].
    *Note: A known pitfall in this implementation is using `abs(minY)` for the z-index calculation, causing vertical misalignment [S1, unofficial].*

4.  **Render/Output:** Iterate $z$ from $0$ to $SAMPLE\_RATE-1$, $y$ from $0$ to $SAMPLE\_RATE-1$, $x$ from $0$ to $SAMPLE\_RATE-1$. If `voxelData[x][y][z]` is true, mark the voxel as occupied [S1, unofficial].
    *Note: Grid allocation uses $SAMPLE\_RATE + 1$ dimensions, but the loop iterates strictly less than $SAMPLE\_RATE$, leaving the outermost layer unrendered [S1, unofficial].*

### Algorithm: Exhaustive Voxel-Face Intersection
This approach iterates over all voxels and checks for intersection with any mesh triangle.

1.  **Define Grid:** Compute voxel size from sensor bounding box extents divided by resolution. Generate coordinate grids using `dr.meshgrid` with 'ij' indexing [S2, unofficial].

2.  **Iterate Voxels:** Initialize a parallel loop over all voxel indices $i$ from $0$ to $height \times width \times depth - 1$. Calculate voxel coordinates:
    $$
    \begin{aligned}
    d &= i \% depth \\
    w &= (i / depth) \% width \\
    h &= (i / depth) / width
    \end{aligned}
    $$
    [S3, unofficial].

3.  **Define Voxel Bounds:** The voxel box is defined by:
    $$
    \begin{aligned}
    min &= (w, h, d) \\
    max &= (w + 1, h + 1, d + 1)
    \end{aligned}
    $$
    [S3, unofficial].

4.  **Check Intersection:** Loop over all faces $f$. Get vertices $v_1, v_2, v_3$. Check intersection:
    $$
    overlap = \text{triangle\_box\_intersection}(min, max, v_1, v_2, v_3)
    $$
    [S3, unofficial].

5.  **Mark Occupancy:** If overlap, set voxel to 1 and break face loop:
    ```python
    if overlap:
        occ[h, w, d] = 1
        break
    ```
    [S3, unofficial].

### Pseudocode (Exhaustive Intersection)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_mesh(vertices, faces, resolution):
    """
    Voxelize a mesh by checking intersection of every voxel AABB with every triangle.
    
    Args:
        vertices: List of (x, y, z) floats.
        faces: List of (i, j, k) integer indices.
        resolution: Tuple (width, height, depth) of integers.
    
    Returns:
        occ: 3D numpy array (depth, height, width) of integers (0 or 1).
    """
    width, height, depth = resolution
    
    # Initialize output grid
    occ = np.zeros((depth, height, width), dtype=np.int32)
    
    # Pre-calculate voxel bounds if needed, or compute on fly
    # For simplicity, we assume unit voxel size in grid coordinates.
    # In practice, 'min' and 'max' in triangle_box_intersection 
    # should be scaled to world space if vertices are in world space.
    
    for i in range(width * height * depth):
        d = i % depth
        w = (i // depth) % width
        h = (i // depth) // width
        
        # Voxel AABB in grid coordinates
        min_coord = np.array([w, h, d], dtype=np.float32)
        max_coord = np.array([w + 1, h + 1, d + 1], dtype=np.float32)
        
        found_intersection = False
        for face in faces:
            v1 = vertices[face[0]]
            v2 = vertices[face[1]]
            v3 = vertices[face[2]]
            
            # Check if triangle intersects voxel AABB
            if triangle_box_intersection(min_coord, max_coord, v1, v2, v3):
                occ[d, h, w] = 1
                found_intersection = True
                break # No need to check other faces for this voxel
        
    return occ

def triangle_box_intersection(box_min, box_max, v1, v2, v3):
    """
    Returns True if the triangle defined by v1, v2, v3 intersects 
    the axis-aligned box defined by box_min and box_max.
    Implementation details depend on the specific intersection algorithm used (e.g., slab method).
    [G] Placeholder for standard AABB-Triangle intersection logic.
    """
    # Standard implementation would go here
    pass
```

### Parameters
*   **SAMPLE_RATE:** Static constant defining the fixed resolution of the voxel grid along each axis. Typical value: 32 [S1, unofficial].
*   **Resolution:** Derived from sensor bounding box extents divided by voxel size [S2, unofficial].

### Edge Cases and Pitfalls
1.  **Index Calculation Error:** In vertex-based voxelization, using `abs(minY)` instead of `abs(minZ)` for the z-index calculation causes vertical misalignment in the voxel grid [S1, unofficial].
2.  **Boundary Rendering:** Grid allocation uses $SAMPLE\_RATE + 1$ dimensions, but the rendering loop iterates strictly less than $SAMPLE\_RATE$, leaving the outermost layer of voxels unrendered [S1, unofficial].
3.  **Face Index 0:** In color-preserving variants, face index 0 is treated as 'not voxelized' space by default. If face 0 overlaps, store `num_faces()` in the voxel instead of 0 to distinguish it from empty space [S3, unofficial].
4.  **Empty Grid Fallback:** If tetravoxelization yields no active voxels (empty model or misaligned AABB), the method falls back to `fillNaive(model)` to prevent empty grids [S4, unofficial].

## Subtopics

### [Adaptive Triangle-to-Voxel Overlap Test](14342-adaptive-triangle-to-voxel-overlap-test.md)
The Adaptive Triangle-to-Voxel Overlap Test is a voxelization algorithm that determines which grid voxels overlap with a mesh triangle by classifying the triangle's axis-aligned bounding box (AABB) thickness into three cases (1D, 2D, or 3D) to optimize the overlap test. It builds on [Occupancy Grid Voxelization](6958-occupancy-grid-voxelization.md) by replacing a naive intersection check with a hierarchical classification that skips geometric tests when the bounding box is thin in multiple dimensions, marking those voxels as fully occupied without further computation. Choose this variant when performance is critical and triangles are large relative to voxels, allowing for early rejection.

### Color-Preserving Occupancy Voxelization
Computes an occupancy grid where voxel values store the index of the first face that intersects the voxel, preserving color information by mapping face indices to voxel values [S3, unofficial]. It differs from binary voxelization by storing `num_faces()` (or the face index) instead of a binary 1. Use this when downstream rendering requires per-voxel color or material information derived from the mesh surface. Pitfall: Face index 0 must be handled specially (e.g., storing `num_faces()`) to avoid ambiguity with empty space [S3, unofficial].

### Tetravoxelization Grid Population
Populates a regular voxel grid by tetravoxelizing input 3D models and mapping binary occupancy results to grid values [S4, unofficial]. It initializes a tetravoxelizer with grid subdivisions, computes voxelization into a 1D array `_voxelOpenGL`, and maps results to a grid structure. Use this when working with hierarchical or specialized voxel formats like tetravoxels. Pitfall: If no active voxels are found, it falls back to `fillNaive` to prevent empty grids [S4, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Vertex-based: $O(N_v)$ to map vertices, plus $O(S^3)$ to render/iterate, where $S$ is `SAMPLE_RATE` [S1, unofficial].
    *   Intersection-based: $O(N_v \cdot N_f \cdot V)$ in the worst case if not optimized, where $V$ is the number of voxels. Parallelization over voxels is common [S3, unofficial].
*   **Memory Complexity:** $O(S^3)$ for the occupancy grid [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Simple to implement; vertex-based method is very fast for sparse meshes; intersection-based method is robust for complex geometry.
*   **Limitations:** Vertex-based method may miss thin geometry not captured by vertices; intersection-based method is computationally expensive for high-resolution grids and dense meshes [S1, unofficial][S3, unofficial].
*   **When to use:** When a coarse binary representation of the mesh is sufficient for optimization or visualization [S2, unofficial].
*   **When not to use:** When high-fidelity surface representation is required, or when the mesh is extremely dense and resolution is high, leading to performance bottlenecks.

## Background
Originated in senior projects and open-source implementations for volumetric rendering and additive manufacturing optimization [S1, unofficial][S2, unofficial]. It is used in basic voxel visualization [S1, unofficial], as an optimization reference for inverse rendering in tomographic volumetric additive manufacturing [S2, unofficial], and in mesh processing pipelines for collision detection or spatial partitioning [S3, unofficial][S4, unofficial].

## Sources
[S1] Dean904/Senior-Project-Volumetric-Rendering, https://github.com/Dean904/Senior-Project-Volumetric-Rendering, fetched 2026-10-26 (unofficial)
[S2] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S3] Mesh Voxelization - GitHub, https://github.com/davidstutz/mesh-voxelization, fetched 2026-10-26 (unofficial)
[S4] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
