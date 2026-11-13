# Signed Distance Field from Mesh

Also called: Signed Mesh SDF, Triangle Mesh SDF, Mesh SDF Rendering

## Overview
This method converts an explicit triangle mesh into a voxelized Signed Distance Field (SDF) where each voxel stores the signed Euclidean distance to the nearest point on the mesh surface [S1, S3, unofficial][S4]. The sign is negative inside the mesh and positive outside, determined by the closest triangle's face normal [S3, unofficial] or a point-in-mesh test [S4]. It differs from [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md) by evaluating distance directly against geometric primitives rather than projecting slices, making it suitable for inputs from upstream reconstruction pipelines where calibrated camera poses are unavailable [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A closed or open triangle mesh (e.g., STL, OBJ) defined by vertices and faces [S1, S3, unofficial].
*   **Output:** A 3D grid or texture where each cell contains a scalar representing the signed distance [S2, unofficial][S4].

**Data Structures:**
1.  **Mesh Representation:**
    *   Vertices: Array of 3D points $(v_0, v_1, \dots)$ [S2, unofficial].
    *   Faces: Array of index triplets $(i, j, k)$ defining triangles [S2, unofficial].
    *   Optional: Axis-Aligned Bounding Box (AABB) tree for acceleration [S4].
2.  **Voxel Grid:**
    *   Dimensions: $R_x \times R_y \times R_z$ (e.g., $50 \times 50 \times 50$ or $50 \times 2500$ for 2D slices) [S2, unofficial].
    *   Layout: Linear array or 3D texture with coordinates mapping to world space $[-1, 1]^3$ via $\text{pos} = (\text{pos} / \text{res}) \times 2.0 - 1.0$ [S2, unofficial].
    *   Storage: Single float per voxel for distance [S2, unofficial].

**Steps:**
1.  **Initialization:** Load mesh into memory, compute global bounds, and initialize the distance field with a large value (e.g., $10000.0$) or infinity [S2, unofficial][S4].
2.  **Voxel Sampling:** For each voxel center $p$ in the grid:
    a.  Find the closest point $cp$ on the mesh surface to $p$ [S4].
    b.  Compute unsigned distance $d = \|p - cp\|$ [S4].
    c.  Determine sign: if the mesh is closed and $p$ is inside, negate $d$ ($d = -d$) [S4]. Alternatively, use the closest triangle's face normal with counter-clockwise winding (outward normal defined by $\text{cross}(v_1-v_0, v_2-v_0)$ pointing out) to assign negative inside and positive outside [S3, unofficial].
    d.  Store signed $d$ in the voxel [S2, unofficial].
3.  **Acceleration (Optional):** Use an AABB tree to quickly find the closest triangle or point, reducing complexity from $O(N \cdot M)$ to $O(N \log M)$, where $N$ is voxels and $M$ is triangles [S4].

**Formulas:**
*   **Triangle Distance (Naive):**
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
*   **Performance:** Naive implementations output `abs(dist)` regardless of inside/outside status, ignoring the sign calculation [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def norm(v):
    return math.sqrt(dot(v, v))

def sub(a, b):
    return (a[0]-b[0], a[1]-b[1], a[2]-b[2])

def add(a, b):
    return (a[0]+b[0], a[1]+b[1], a[2]+b[2])

def scale(v, s):
    return (v[0]*s, v[1]*s, v[2]*s)

def dist_to_triangle(p, v0, v1, v2):
    e1 = sub(v1, v0)
    e2 = sub(v2, v0)
    v0p = sub(p, v0)
    n = cross(e1, e2)
    n_len = norm(n)
    if n_len < 1e-8:
        return float('inf'), (0.0, 0.0, 0.0)
    n = scale(n, 1.0/n_len)
    ba = sub(v0, v1)
    pa = sub(v0, p)
    proj = dot(ba, (ba[0]*max(0, min(1, dot(ba, pa)/dot(ba, ba))), 
                     ba[1]*max(0, min(1, dot(ba, pa)/dot(ba, ba))), 
                     ba[2]*max(0, min(1, dot(ba, pa)/dot(ba, ba))))) - dot(pa, ba)
    if dot(ba, ba) < 1e-8:
        return float('inf'), n
    d_plane = abs(dot(v0p, n))
    return d_plane, n

def ray_triangle_intersect(origin, direction, v0, v1, v2):
    e1 = sub(v1, v0)
    e2 = sub(v2, v0)
    pvec = cross(direction, e2)
    det = dot(e1, pvec)
    if abs(det) < 1e-8:
        return False
    inv_det = 1.0 / det
    tvec = sub(origin, v0)
    u = dot(tvec, pvec) * inv_det
    if u < 0 or u > 1:
        return False
    qvec = cross(tvec, e1)
    v = dot(direction, qvec) * inv_det
    if v < 0 or u + v > 1:
        return False
    t = dot(e2, qvec) * inv_det
    return t > 1e-8

def is_point_inside_mesh(p, vertices, faces):
    ray_dir = (1.0, 0.0, 0.0)
    intersections = 0
    for face in faces:
        v0, v1, v2 = vertices[face[0]], vertices[face[1]], vertices[face[2]]
        if ray_triangle_intersect(p, ray_dir, v0, v1, v2):
            intersections += 1
    return intersections % 2 == 1

def compute_mesh_sdf(vertices, faces, grid_bounds, resolution, is_closed=False):
    min_pt, max_pt = grid_bounds
    voxel_size = (max_pt[0]-min_pt[0])/resolution, (max_pt[1]-min_pt[1])/resolution, (max_pt[2]-min_pt[2])/resolution
    sdf_grid = [[[float('inf') for _ in range(resolution)] for _ in range(resolution)] for _ in range(resolution)]
    
    for z in range(resolution):
        for y in range(resolution):
            for x in range(resolution):
                p = add(add(min_pt, scale((x, y, z), voxel_size)), scale(voxel_size, 0.5))
                min_dist = float('inf')
                closest_normal = (0.0, 0.0, 0.0)
                
                for face in faces:
                    v0, v1, v2 = vertices[face[0]], vertices[face[1]], vertices[face[2]]
                    d, n = dist_to_triangle(p, v0, v1, v2)
                    if d < min_dist:
                        min_dist = d
                        closest_normal = n
                
                sign = 1.0
                if is_closed:
                    if is_point_inside_mesh(p, vertices, faces):
                        sign = -1.0
                    else:
                        if dot(closest_normal, sub(p, (v0[0]+v1[0]+v2[0])/3.0, (v0[1]+v1[1]+v2[1])/3.0, (v0[2]+v1[2]+v2[2])/3.0)) < 0:
                            sign = -1.0
                sdf_grid[z][y][x] = sign * min_dist
    return sdf_grid
```

## Subtopics

### [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md)
This method converts explicit geometry, such as triangle meshes or oriented point clouds, into a Truncated Signed Distance Field (TSDF) voxel grid by computing the closest-surface distance for each voxel center and assigning a sign based on surface normals or an offset fallback. It differs from [Projective TSDF Fusion](3649-mesh-to-tsdf-sequence-encoding-pipeline.md) by operating directly on geometric primitives rather than depth images, making it suitable for inputs from upstream reconstruction pipelines where calibrated camera poses are unavailable or unnecessary.

### [SDF Dataset Normalization](3787-sdf-dataset-normalization.md)
SDF Dataset Normalization is a preprocessing step that transforms the vertex coordinates of an input 3D mesh so that the geometry fits within a standardized canonical space, typically the unit cube $[-1, 1]^3$ centered at the origin. This method solves the problem of inconsistent scale and position in raw mesh data, which is critical for downstream Signed Distance Field (SDF) generation where grid resolution and sampling bounds depend on predictable input dimensions. It differs from [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md) and [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md) by operating directly on the source mesh geometry prior to discretization, ensuring the input data is normalized before voxelization.

### [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md)
A method that converts a triangle mesh into a sparse, tree-structured signed or unsigned distance grid using OpenVDB, storing only active voxels near the surface for memory efficiency. It differs from dense voxelization by dynamically allocating memory only for the narrow band around the mesh, making it suitable for large or complex models where ray-marched rendering or fast implicit queries are required.

### [Naive SDF Generation (CPU)](10688-naive-sdf-generation-cpu.md)
Naive SDF Generation (CPU) computes the Signed Distance Field (SDF) for a triangle mesh by explicitly evaluating the distance from every voxel sample point to every triangle in the mesh. It converts an explicit mesh representation into an implicit volume representation suitable for ray marching or voxel-based rendering [G]. Unlike spatial acceleration methods, it performs a linear scan of all triangles for each sample, making it simple to implement but computationally prohibitive for high-resolution volumes or complex meshes.

### Mesh to SDF Volume Conversion
A technique to convert a manifold 3D mesh into a Signed Distance Field (SDF) volume stored in an atlas texture using a utility sampler [S5, unofficial]. It involves creating a sampler function using `VolumeSamplers.createMeshInstanceSdfSampler(mesh)`, initializing the volume renderer's atlas texture with specific resolution, origin, and voxel size using `volumeRenderer.createAtlasTexture(resolution, origin, voxelSize, timeCount)`, and updating the atlas texture by calling `volumeRenderer.updateAtlasTexture` with a callback that invokes the mesh sampler with world coordinates: `(xi, yi, zi, x, y, z, t) => sampler(x, y, z)` [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Naive implementation is $O(N \cdot M)$, where $N$ is the number of voxels and $M$ is the number of triangles [S1, unofficial]. With AABB acceleration, it approaches $O(N \log M)$ [S4].
*   **Memory Complexity:** $O(N)$ for the dense grid [G]. Sparse variants use $O(K)$ where $K$ is the number of active voxels near the surface [S1, unofficial].
*   **Performance:** Suitable for moderate-resolution grids and simple meshes [S2, unofficial]. High-resolution grids or complex meshes require acceleration structures or GPU parallelization [G].
*   **Strengths:** Robust for rendering CAD meshes and implicit surfaces; supports CSG operations; differentiable [S1, unofficial].
*   **Limitations:** Open meshes cause artifacts (skirts); sharp corners may have sign errors; naive implementation is slow [S3, unofficial][S1, unofficial].
*   **When to Use:** When rendering explicit meshes as implicit surfaces, or when inputs are from reconstruction pipelines without camera poses [S1, unofficial][S3, unofficial].
*   **When Not to Use:** For very large, complex meshes where memory is constrained (use Sparse VDB) or when camera poses are available and depth-based fusion is preferred [G].

## Background
The method originates from the need to render triangle meshes as implicit surfaces, enabling robust interval-arithmetic bisection and CSG operations [S1, unofficial]. It is used in libraries like `isoray` for rendering CAD meshes (e.g., brake calipers, hooks) [S1, unofficial]. CGAL's Isosurfacing_3 module also provides tools for generating signed offset fields from closed surface meshes using AABB trees [S4]. The technique is fundamental in 3D reconstruction pipelines where explicit geometry needs to be converted to volumetric representations for rendering or further processing [S1, unofficial][S5, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
[S4] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S5] here, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
