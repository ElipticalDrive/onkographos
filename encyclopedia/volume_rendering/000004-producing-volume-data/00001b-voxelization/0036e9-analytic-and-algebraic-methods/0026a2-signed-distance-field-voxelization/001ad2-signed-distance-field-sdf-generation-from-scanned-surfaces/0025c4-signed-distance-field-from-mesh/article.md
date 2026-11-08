# Signed Distance Field from Mesh

Also called: Signed Mesh SDF, Triangle Mesh SDF, Mesh SDF Rendering

## Overview
This method converts an explicit triangle mesh into a voxelized Signed Distance Field (SDF) where each voxel stores the signed Euclidean distance to the nearest point on the mesh surface [S1, S3, unofficial][S4]. The sign is negative inside the mesh and positive outside, determined by the closest triangle's face normal or a point-in-mesh test [S3, unofficial][S4]. It differs from [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md) by evaluating distance directly against geometric primitives (triangles) rather than projecting slices, making it suitable for inputs from upstream reconstruction pipelines where calibrated camera poses are unavailable [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A closed or open triangle mesh (e.g., STL, OBJ) defined by vertices and faces.
*   **Output:** A 3D grid (texture or volume) where each cell contains a scalar representing the signed distance.

**Data Structures:**
1.  **Mesh Representation:**
    *   Vertices: Array of 3D points $(v_0, v_1, \dots)$.
    *   Faces: Array of indices $(i, j, k)$ defining triangles.
    *   Optional: Axis-Aligned Bounding Box (AABB) tree for acceleration [S4].
2.  **Voxel Grid:**
    *   Dimensions: $R_x \times R_y \times R_z$ (e.g., $50 \times 50 \times 50$ or $50 \times 2500$ for 2D slices) [S2, unofficial].
    *   Layout: Linear array or 3D texture. Coordinates map to world space $[-1, 1]^3$ or custom bounds [S2, unofficial].
    *   Storage: Single float per voxel for distance [S2, unofficial].

**Steps:**
1.  **Initialization:** Load mesh into memory. Compute global bounds. Initialize distance field with a large value (e.g., $10000.0$) or infinity [S2, unofficial][S4].
2.  **Voxel Sampling:** For each voxel center $p$ in the grid:
    a.  Find the closest point $cp$ on the mesh surface to $p$.
    b.  Compute unsigned distance $d = \|p - cp\|$.
    c.  Determine sign:
        *   If mesh is closed and $p$ is inside, $d = -d$.
        *   Sign determination methods:
            *   **Normal-based:** Use the normal of the closest triangle. If the mesh is closed with counter-clockwise winding (outward normal defined by $\text{cross}(v_1-v_0, v_2-v_0)$ pointing out), the sign is negative if $p$ is on the inner side of the closest face [S3, unofficial].
            *   **Point-in-Mesh:** Use a ray-casting or winding number test to determine if $p$ is inside the mesh [S4].
    d.  Store signed $d$ in the voxel.
3.  **Acceleration (Optional):** Use an AABB tree to quickly find the closest triangle or point, reducing complexity from $O(N \cdot M)$ to $O(N \log M)$, where $N$ is voxels and $M$ is triangles [S4].

**Formulas:**
*   **Triangle Distance (Unoptimized/Naive):**
    For a triangle with vertices $a, b, c$ and query point $p$:
    Let $\mathbf{ba} = a - b$, $\mathbf{pa} = a - p$, $\mathbf{n} = \text{normalize}(\text{cross}(b-a, c-a))$.
    Inside check: $\sum \text{sign}(\mathbf{ba} \times \mathbf{n} \cdot \mathbf{pa}) < 2$ [S2, unofficial].
    Projection: $\mathbf{proj} = \mathbf{ba} \cdot \text{clamp}(\frac{\mathbf{ba} \cdot \mathbf{pa}}{\mathbf{ba} \cdot \mathbf{ba}}, 0, 1) - \mathbf{pa}$ [S2, unofficial].
    Distance $d = \sqrt{\min(\dots)}$ if outside, else $d = \frac{(\mathbf{n} \cdot \mathbf{pa})^2}{\mathbf{n} \cdot \mathbf{n}}$ [S2, unofficial].
*   **Closest Point (AABB Tree):**
    $cp = \text{tree.closest\_point}(p)$ [S4].
    $d = \sqrt{(p - cp)^2}$ [S4].
*   **Sign Determination (CGAL):**
    If `is_closed` and `sotm(p) == ON_BOUNDED_SIDE`, then $d = -d$ [S4].

**Parameters:**
*   **Resolution:** Grid dimensions (e.g., 50) [S2, unofficial].
*   **Truncation Distance:** For TSDF variants, max distance to store [S3, unofficial].
*   **Is Closed:** Boolean flag indicating if the mesh is watertight, required for accurate inside/outside determination [S4].

**Edge Cases and Pitfalls:**
*   **Open Meshes:** Grow a "skirt" up to `trunc_dist` past their rim if signed mode is used [S3, unofficial].
*   **Winding Order:** Mesh must be counter-clockwise from outside. Inside-out meshes remain inside-out [S3, unofficial].
*   **Sharp Corners:** Edges/corners with faces meeting at under 90 degrees may take the wrong side due to nearest-face tie-breaking by lowest index [S3, unofficial].
*   **Precision:** Rounding errors can cause incorrect sign assignment at specific geometric alignments (e.g., 90-degree edge off-grid axes) [S3, unofficial].
*   **Performance:** Naive implementation is $O(N \cdot M)$, prohibitive for high-resolution volumes or complex meshes [S10688].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mesh_sdf(mesh, grid_bounds, resolution):
    """
    Compute Signed Distance Field from a triangle mesh.
    
    Args:
        mesh: Object containing vertices (N, 3) and faces (M, 3).
        grid_bounds: Tuple of (min_point, max_point) defining the voxel grid extent.
        resolution: Integer, number of voxels per axis (assuming cubic grid).
        
    Returns:
        sdf_grid: 3D numpy array of shape (resolution, resolution, resolution).
    """
    # Initialize grid with large distance
    sdf_grid = np.full((resolution, resolution, resolution), np.inf)
    
    # Compute voxel size and origin
    min_pt, max_pt = grid_bounds
    voxel_size = (max_pt - min_pt) / resolution
    
    # Precompute mesh data for fast access
    vertices = mesh.vertices
    faces = mesh.faces
    
    # Optional: Build AABB tree for acceleration [S4]
    # tree = build_aabb_tree(mesh)
    
    for z in range(resolution):
        for y in range(resolution):
            for x in range(resolution):
                # Calculate voxel center in world space
                p = min_pt + np.array([x, y, z]) * voxel_size + voxel_size / 2.0
                
                # Find closest point on mesh
                # Method 1: Naive scan [S2, S10688]
                min_dist = np.inf
                closest_normal = None
                
                for face in faces:
                    v0, v1, v2 = vertices[face]
                    # Compute distance from p to triangle v0-v1-v2
                    d, normal = distance_to_triangle(p, v0, v1, v2)
                    if d < min_dist:
                        min_dist = d
                        closest_normal = normal
                
                # Method 2: Accelerated [S4]
                # cp = tree.closest_point(p)
                # min_dist = np.linalg.norm(p - cp)
                # closest_normal = compute_normal_at_closest_point(p, mesh)
                
                # Determine sign
                sign = 1.0
                if mesh.is_closed:
                    # Check if point is inside mesh
                    # Using ray casting or winding number
                    if is_point_inside_mesh(p, mesh):
                        sign = -1.0
                    else:
                        # Fallback: use closest face normal if inside test is expensive/unreliable
                        # Note: This assumes outward normals [S3, unofficial]
                        # If normal points away from p, p is outside (sign +1)
                        # If normal points towards p, p is inside (sign -1)
                        # Vector from surface to point
                        # This logic is simplified; robust implementation uses sotm [S4]
                        pass 
                
                # Apply sign
                signed_dist = sign * min_dist
                sdf_grid[z, y, x] = signed_dist
                
    return sdf_grid

def distance_to_triangle(p, v0, v1, v2):
    """
    Compute distance from point p to triangle v0-v1-v2.
    Returns (distance, normal_at_closest_point).
    """
    # Vector from v0 to v1, v0 to v2
    e1 = v1 - v0
    e2 = v2 - v0
    # Vector from v0 to p
    v0p = p - v0
    
    # Normal of the triangle
    normal = np.cross(e1, e2)
    normal_len = np.linalg.norm(normal)
    if normal_len < 1e-8:
        return np.inf, np.array([0, 0, 0]) # Degenerate triangle
    normal = normal / normal_len
    
    # Project p onto the plane
    # Distance to plane
    dist_to_plane = np.abs(np.dot(v0p, normal))
    
    # Check if projection is inside triangle
    # Barycentric coordinates or edge tests
    # Edge tests: cross(v1-v0, p-v0) . normal >= 0, etc.
    # Simplified: use closest point on triangle algorithm
    
    # Closest point on triangle
    # Using barycentric coordinates
    d1 = np.dot(e1, e1)
    d2 = np.dot(e2, e2)
    d3 = np.dot(e1, e2)
    d = d1 * d2 - d3 * d3
    s = np.dot(v0p, e1) * d2 - np.dot(v0p, e2) * d3
    t = np.dot(v0p, e2) * d1 - np.dot(v0p, e1) * d3
    
    if d < 1e-8:
        return np.inf, normal
        
    s /= d
    t /= d
    
    if s <= 0 and t <= 0:
        # Closest to v0
        closest = v0
    elif s >= 1 and t <= 0:
        # Closest to v1
        closest = v1
    elif t >= 1 and s <= 0:
        # Closest to v2
        closest = v2
    else:
        # Inside or on edge
        # Clamp s and t
        if s + t > 1:
            if s > t:
                closest = v1
            else:
                closest = v2
        else:
            closest = v0 + s * e1 + t * e2
            
    dist = np.linalg.norm(p - closest)
    return dist, normal

def is_point_inside_mesh(p, mesh):
    """
    Ray casting algorithm to determine if point p is inside mesh.
    """
    # Cast ray in arbitrary direction (e.g., +X)
    ray_dir = np.array([1, 0, 0])
    ray_origin = p
    
    intersections = 0
    for face in mesh.faces:
        v0, v1, v2 = mesh.vertices[face]
        if ray_triangle_intersect(ray_origin, ray_dir, v0, v1, v2):
            intersections += 1
            
    return intersections % 2 == 1

def ray_triangle_intersect(origin, direction, v0, v1, v2):
    """
    Möller-Trumbore intersection algorithm [S2, unofficial].
    """
    e1 = v1 - v0
    e2 = v2 - v0
    pvec = np.cross(direction, e2)
    det = np.dot(e1, pvec)
    
    if abs(det) < 1e-8:
        return False # Parallel
        
    inv_det = 1.0 / det
    tvec = origin - v0
    u = np.dot(tvec, pvec) * inv_det
    
    if u < 0 or u > 1:
        return False
        
    qvec = np.cross(tvec, e1)
    v = np.dot(direction, qvec) * inv_det
    
    if v < 0 or u + v > 1:
        return False
        
    t = np.dot(e2, qvec) * inv_det
    
    if t > 1e-8:
        return True
        
    return False
```

## Subtopics

### [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md)
This method converts explicit geometry, such as triangle meshes or oriented point clouds, into a Truncated Signed Distance Field (TSDF) voxel grid by computing the closest-surface distance for each voxel center and assigning a sign based on surface normals or an offset fallback. It differs from [Projective TSDF Fusion](3649-mesh-to-tsdf-sequence-encoding-pipeline.md) by operating directly on geometric primitives rather than depth images, making it suitable for inputs from upstream reconstruction pipelines where calibrated camera poses are unavailable or unnecessary.

### [SDF Dataset Normalization](3787-sdf-dataset-normalization.md)
SDF Dataset Normalization is a preprocessing method that transforms a 3D mesh's vertex coordinates so that the geometry fits within a standardized canonical space, typically a unit cube or unit sphere centered at the origin. This process solves the problem of inconsistent scale and position in input meshes, which is critical for downstream volume rendering pipelines such as Signed Distance Field (SDF) generation, where grid resolution and sampling bounds depend on predictable input dimensions. It differs from [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md) and [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md) by operating on the source mesh geometry directly rather than on an existing voxel grid, ensuring the input data is normalized before discretization.

### [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md)
A method that converts a triangle mesh into a sparse, tree-structured signed or unsigned distance grid using OpenVDB, storing only active voxels near the surface for memory efficiency. It differs from dense voxelization by dynamically allocating memory only for the narrow band around the mesh, making it suitable for large or complex models where ray-marched rendering or fast implicit queries are required.

### [Naive SDF Generation (CPU)](10688-naive-sdf-generation-cpu.md)
Naive SDF Generation (CPU) computes the Signed Distance Field (SDF) for a triangle mesh by explicitly evaluating the distance from every voxel sample point to every triangle in the mesh. It solves the problem of converting an explicit mesh representation into an implicit volume representation suitable for ray marching or voxel-based rendering. Unlike spatial acceleration methods, it performs a linear scan of all triangles for each sample, making it simple to implement but computationally prohibitive for high-resolution volumes or complex meshes.

### Mesh to SDF Volume Conversion
A technique to convert a manifold 3D mesh into a Signed Distance Field (SDF) volume stored in an atlas texture using a utility sampler [S5, unofficial]. It involves creating a sampler function using `VolumeSamplers.createMeshInstanceSdfSampler(mesh)`, initializing the volume renderer's atlas texture with specific resolution, origin, and voxel size using `volumeRenderer.createAtlasTexture(resolution, origin, voxelSize, timeCount)`, and updating the atlas texture by calling `volumeRenderer.updateAtlasTexture` with a callback that invokes the mesh sampler with world coordinates: `(xi, yi, zi, x, y, z, t) => sampler(x, y, z)` [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Naive implementation is $O(N \cdot M)$, where $N$ is the number of voxels and $M$ is the number of triangles [S10688]. With AABB acceleration, it approaches $O(N \log M)$ [S4].
*   **Memory Complexity:** $O(N)$ for the dense grid. Sparse variants use $O(K)$ where $K$ is the number of active voxels near the surface [S4498].
*   **Performance:** Suitable for moderate-resolution grids and simple meshes. High-resolution grids or complex meshes require acceleration structures or GPU parallelization [S2, unofficial].
*   **Strengths:** Robust for rendering CAD meshes and implicit surfaces; supports CSG operations; differentiable [S1, unofficial].
*   **Limitations:** Open meshes cause artifacts (skirts); sharp corners may have sign errors; naive implementation is slow [S3, S10688].
*   **When to Use:** When rendering explicit meshes as implicit surfaces, or when inputs are from reconstruction pipelines without camera poses [S1, S3, unofficial].
*   **When Not to Use:** For very large, complex meshes where memory is constrained (use Sparse VDB) or when camera poses are available and depth-based fusion is preferred [S3649].

## Background
The method originates from the need to render triangle meshes as implicit surfaces, enabling robust interval-arithmetic bisection and CSG operations [S1, unofficial]. It is used in libraries like `isoray` for rendering CAD meshes (e.g., brake calipers, hooks) [S1, unofficial]. CGAL's Isosurfacing_3 module also provides tools for generating signed offset fields from closed surface meshes using AABB trees [S4]. The technique is fundamental in 3D reconstruction pipelines where explicit geometry needs to be converted to volumetric representations for rendering or further processing [S1, S5, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
[S4] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S5] here, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
