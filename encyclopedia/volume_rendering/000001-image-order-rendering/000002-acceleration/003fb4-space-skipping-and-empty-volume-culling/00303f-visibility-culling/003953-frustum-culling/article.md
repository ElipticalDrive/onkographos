# Frustum Culling

Also called: ToF View Frustum Culling, View Frustum Culling, markVisible

## Overview
Frustum culling is a visibility culling technique that avoids rendering geometry located outside the camera's field of view, thereby eliminating unnecessary computation [S2, S4, unofficial]. It distinguishes itself from occlusion culling by relying solely on geometric containment relative to the viewing frustum planes, rather than depth-buffered visibility tests [S2, unofficial]. It serves as a coarse-grained filter, often preceding more expensive fine-grained culling methods [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of geometric primitives (points, voxels, blocks, or segments) defined by their spatial extents (e.g., bounding spheres or boxes); the camera's view matrix, projection matrix, near plane distance ($n$), and far plane distance ($f$).
*   **Outputs:** A boolean mask or subset of primitives indicating which elements intersect the viewing frustum [S1, S2, S4, unofficial].

**Data Structures:**
Primitives are typically summarized by conservative bounding volumes to allow for fast intersection tests. Common structures include:
*   **Bounding Sphere:** Defined by center $c_k = (x_k, y_k, z_k)$ and radius $r_k$.
*   **Axis-Aligned Bounding Box (AABB):** Defined by minimum and maximum corners.
*   **Hierarchical Spatial Partition:** Nodes in a tree (e.g., Octree, BVH) where each node stores a bounding volume for its subtree [S2, S4, unofficial].

**Algorithm Steps:**
1.  **Frustum Plane Extraction:** Extract the six planes of the viewing frustum (left, right, bottom, top, near, far) from the combined view-projection matrix. Each plane is represented by a normal vector $\mathbf{n}_i$ and a distance $d_i$ from the origin, such that a point $\mathbf{p}$ is inside if $\mathbf{n}_i \cdot \mathbf{p} + d_i \ge 0$ [S2, S4, unofficial].
2.  **Bounding Volume Test:** For each primitive or node, test its bounding volume against the six frustum planes.
    *   **Sphere Test:** A sphere with center $c_k$ and radius $r_k$ is **outside** the frustum if, for any plane $i$, the signed distance from the center to the plane is less than the negative radius:
        $$ \mathbf{n}_i \cdot c_k + d_i < -r_k $$
        If this condition holds for any plane, the sphere is culled [S4, unofficial].
    *   **Box Test:** An AABB is outside if any of its eight corners lies outside any of the six planes. Alternatively, check if the box's extent along the plane normal is less than the plane's distance offset [S2, unofficial].
3.  **Hierarchical Propagation (Optional):** If using a spatial hierarchy, if a node is completely outside the frustum, all its descendants are marked as invisible and skipped. If a node is completely inside, all descendants are marked visible [S2, unofficial].
4.  **Result Compilation:** Return the set of primitives/nodes that passed the intersection test [S1, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_sphere_in_frustum(sphere_center, sphere_radius, frustum_planes):
    """
    sphere_center: tuple (x, y, z)
    sphere_radius: float
    frustum_planes: list of 6 tuples (nx, ny, nz, d)
    Returns: True if sphere intersects frustum, False if completely outside.
    """
    for (nx, ny, nz, d) in frustum_planes:
        # Signed distance from sphere center to plane
        dist = nx * sphere_center[0] + ny * sphere_center[1] + nz * sphere_center[2] + d
        # Cull if distance is less than negative radius
        if dist < -sphere_radius:
            return False
    return True

def frustum_cull_primitives(primitives, view_matrix, proj_matrix, near, far):
    """
    primitives: list of objects with 'center' (x,y,z) and 'radius'
    view_matrix: 4x4 matrix
    proj_matrix: 4x4 matrix
    near: float
    far: float
    Returns: list of primitives that are visible
    """
    # Extract frustum planes from view-projection matrix
    # Note: Standard extraction logic depends on coordinate system (LH/RH)
    # Assuming standard OpenGL-style extraction for Left, Right, Bottom, Top, Near, Far
    frustum_planes = extract_frustum_planes(view_matrix, proj_matrix, near, far)
    
    visible_primitives = []
    for prim in primitives:
        if is_sphere_in_frustum(prim.center, prim.radius, frustum_planes):
            visible_primitives.append(prim)
            
    return visible_primitives

def extract_frustum_planes(view_matrix, proj_matrix, near, far):
    """
    Extracts 6 planes from combined VP matrix.
    Returns list of (nx, ny, nz, d) for Left, Right, Bottom, Top, Near, Far.
    Implementation detail: Standard matrix plane extraction.
    """
    # Combine matrices
    vp_matrix = multiply_matrices(view_matrix, proj_matrix)
    
    # Extract planes from VP matrix columns
    # Left plane: vp[3] + vp[0]
    # Right plane: vp[3] - vp[0]
    # Bottom plane: vp[3] + vp[1]
    # Top plane: vp[3] - vp[1]
    # Near plane: vp[3] + vp[2]
    # Far plane: vp[3] - vp[2]
    
    planes = []
    
    # Left
    p = [vp_matrix[3][0] + vp_matrix[0][0], vp_matrix[3][1] + vp_matrix[0][1], vp_matrix[3][2] + vp_matrix[0][2], vp_matrix[3][3] + vp_matrix[0][3]]
    planes.append(normalize_plane(p))
    
    # Right
    p = [vp_matrix[3][0] - vp_matrix[0][0], vp_matrix[3][1] - vp_matrix[0][1], vp_matrix[3][2] - vp_matrix[0][2], vp_matrix[3][3] - vp_matrix[0][3]]
    planes.append(normalize_plane(p))
    
    # Bottom
    p = [vp_matrix[3][0] + vp_matrix[1][0], vp_matrix[3][1] + vp_matrix[1][1], vp_matrix[3][2] + vp_matrix[1][2], vp_matrix[3][3] + vp_matrix[1][3]]
    planes.append(normalize_plane(p))
    
    # Top
    p = [vp_matrix[3][0] - vp_matrix[1][0], vp_matrix[3][1] - vp_matrix[1][1], vp_matrix[3][2] - vp_matrix[1][2], vp_matrix[3][3] - vp_matrix[1][3]]
    planes.append(normalize_plane(p))
    
    # Near
    p = [vp_matrix[3][0] + vp_matrix[2][0], vp_matrix[3][1] + vp_matrix[2][1], vp_matrix[3][2] + vp_matrix[2][2], vp_matrix[3][3] + vp_matrix[2][3]]
    planes.append(normalize_plane(p))
    
    # Far
    p = [vp_matrix[3][0] - vp_matrix[2][0], vp_matrix[3][1] - vp_matrix[2][1], vp_matrix[3][2] - vp_matrix[2][2], vp_matrix[3][3] - vp_matrix[2][3]]
    planes.append(normalize_plane(p))
    
    return planes

def normalize_plane(plane):
    """
    Normalizes the plane vector (nx, ny, nz) and scales d accordingly.
    """
    length = (plane[0]**2 + plane[1]**2 + plane[2]**2)**0.5
    if length == 0:
        return (0, 0, 0, 0)
    return (plane[0]/length, plane[1]/length, plane[2]/length, plane[3]/length)
```

**Parameters and Edge Cases:**
*   **Bounding Volume Tightness:** Loose bounding volumes (e.g., spheres) may admit primitives that are technically outside the frustum but intersect the bounding volume, leading to false positives. Tighter bounds (e.g., OBB) reduce false positives but increase test cost [S4, unofficial].
*   **Precision:** Floating-point precision errors in plane extraction or distance calculation can cause borderline cases to be incorrectly culled or retained. [G] Ensure plane normals are normalized before testing.
*   **Degenerate Input:** If the camera is inside a bounding volume, the standard sphere test still holds, but hierarchical culling must handle the "inside" case correctly to avoid culling the containing node [S2, unofficial].

## Subtopics

### View Frustum Culling for Segments
An optimization for segment-based tetrahedral meshing where segments lying outside the view frustum are rejected early in the rendering pipeline. This prevents unnecessary refinement caused by DAG dependencies, which would otherwise increase mesh size and computational load for regions not visible to the camera [S3].

### Two-Level Visibility Culling
A hierarchical approach used in out-of-core rendering (e.g., TideGS) that separates culling into CPU and GPU stages. Level 1 (CPU) uses coarse bounding spheres to compute an active block set $\mathcal{K}_t$ via a 6-plane frustum test, filtering blocks before GPU transfer. Level 2 (GPU) materializes these blocks and applies exact 3DGS projection/rasterization to determine the final contributing set $I_t$. Level 1 is conservative (may admit extra blocks), while Level 2 ensures exact rendering semantics [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ primitives, where each primitive requires a constant number of plane tests (6 for frustum). With hierarchical structures, complexity becomes $O(K)$ where $K$ is the number of leaf nodes visited, significantly reducing work for sparse scenes.
*   **Memory Complexity:** $O(1)$ auxiliary memory per primitive for bounding volume storage. Hierarchical structures require $O(M)$ memory for $M$ nodes.
*   **Performance:** Highly efficient for large datasets where a significant portion of geometry is outside the view frustum. It is often the first stage in a culling pipeline due to its low computational cost [S2, S4, unofficial].
*   **Strengths:** Simple to implement, robust, and effective at reducing the number of primitives processed by subsequent, more expensive stages (e.g., rasterization, ray marching) [S2, S4, unofficial].
*   **Limitations:** Does not account for occlusion by other geometry; primitives inside the frustum may still be hidden [S2, unofficial]. Conservative bounding volumes can lead to false positives, reducing culling efficiency [S4, unofficial].

## Background
Frustum culling is a fundamental technique in computer graphics, originally developed to optimize the rendering pipeline by avoiding the processing of geometry outside the camera's field of view [S2, unofficial]. It is widely supported in graphics libraries (e.g., OpenGL's back-face culling flag is related, though distinct) and is a standard component of modern rendering engines [S2, unofficial]. In the context of volume rendering and large-scale datasets, it is often combined with hierarchical spatial partitioning to accelerate culling [S2, unofficial]. Recent applications include out-of-core rendering of large-scale 3D Gaussian Splatting scenes, where it is used to filter blocks before GPU transfer [S4, unofficial].

## Sources
[S1] brownvc/gftorf, https://github.com/brownvc/gftorf, fetched 2026-10-27 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
[S3] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
[S4] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
