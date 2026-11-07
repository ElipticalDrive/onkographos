# Signed Distance Field from Mesh

Also called: Signed Mesh SDF, Triangle Mesh SDF, Mesh SDF Rendering

## Overview
This method converts a triangulated surface mesh into a 3D distance field by computing the Euclidean distance from each voxel center to the closest point on the mesh surface, with sign assignment based on surface normals or topological containment [S1, unofficial][S3, unofficial][S4]. It enables rendering and isosurfacing of explicit CAD or scanned meshes using implicit surface techniques, such as interval-arithmetic bisection or marching cubes [S1, unofficial][S4]. Unlike [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md), which processes depth slices, this method operates directly on geometric primitives (triangles) [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A triangulated mesh (e.g., STL format) defined by vertices and face indices [S1, unofficial][S4].
- **Output:** A scalar field (texture or grid) where each voxel stores the distance to the mesh surface [S2, unofficial][S4]. Note that some GPU implementations output the absolute distance `abs(dist)` regardless of inside/outside status [S2, unofficial], while CPU implementations compute a signed field [S3, unofficial][S4].

**Data Structures:**
1.  **Mesh Representation:** Vertices and faces. For GPU implementations, vertices are stored in a texture (`meshtex`) with a layout of 3 floats (RGB) per texel and a stride of 16384 texels per row [S2, unofficial]. For CPU implementations, an Accelerated Bounding Box Tree (`AABB_tree`) is built from mesh faces using `AABB_face_graph_triangle_primitive` and `AABB_traits_3` [S4].
2.  **Voxel Grid:** A 3D grid of resolution $R \times R_2$ (e.g., $50 \times 2500$) mapping coordinates to space $[-1, 1]^3$ [S2, unofficial].
3.  **Oracle/Integrator:** A structure holding the mesh tree, a boolean flag `is_closed`, and a `Side_of_triangle_mesh` functor to determine sign logic [S4].

**Numbered Steps:**
1.  **Initialization:** Load the mesh file (e.g., STL) into a `MeshSDF` object or build the AABB tree [S1, unofficial][S4]. Compute the bounding box and map coordinates to $[-1, 1]^3$ using $pos = (coord / res) * 2.0 - 1.0$ [S2, unofficial].
2.  **Voxel Iteration:** For each voxel center $p$ in the grid:
    a. Map grid coordinates to world space $[-1, 1]^3$: $pos = (coord / res) * 2.0 - 1.0$ [S2, unofficial].
    b. Find the closest point $cp$ on the mesh surface to $p$.
        - *GPU Approach:* Iterate through all triangles. For each triangle with vertices $a, b, c$, compute the distance $d$ to the triangle using a point-triangle distance function (`udTriangle`) [S2, unofficial].
        - *CPU Approach:* Query the AABB tree: $cp = tree.closest_point(p)$ [S4].
    c. Compute the unsigned Euclidean distance: $d_{unsigned} = \|p - cp\|$ [S2, unofficial][S4].
    d. Determine the sign:
        - *Normal-based:* If the mesh is closed and has consistent winding (counter-clockwise from outside), the sign is determined by the closest triangle's face normal $\mathbf{n}$. If $p$ is on the "inside" side of the normal, the sign is negative [S3, unofficial].
        - *Topological:* If `is_closed` is true, use a side-of-triangle-mesh functor (`Side_of_triangle_mesh`) to check if $p$ is on the bounded side. If so, negate the distance: $d = -d_{unsigned}$ [S4].
    e. Store the signed distance in the output grid/texture [S2, unofficial][S4].

**Formulas:**
*   **Point-Triangle Distance (Outside):**
    $$d = \sqrt{\min(\dots)}$$
    The specific minimum is taken over distances to edges and vertices if the projection falls outside the triangle [S2, unofficial].
*   **Point-Triangle Distance (Inside/Projection):**
    If the point projects onto the triangle plane, the distance is the perpendicular distance to the plane:
    $$d = \frac{(\mathbf{n} \cdot \mathbf{pa})^2}{\mathbf{n} \cdot \mathbf{n}}$$
    Where $\mathbf{n}$ is the triangle normal and $\mathbf{pa}$ is the vector from a triangle vertex to the query point [S2, unofficial][G].
*   **Inside Check (Winding Number Approximation):**
    $$\sum \text{sign}(\mathbf{ba} \times \mathbf{n} \cdot \mathbf{pa}) < 2$$
    Used to determine if a point is inside a triangle for distance calculation [S2, unofficial].
*   **Ray-Triangle Intersection (Möller-Trumbore):**
    Used for raycasting approaches or validation:
    $$\mathbf{e1} = \mathbf{v1}-\mathbf{v0}, \quad \mathbf{e2} = \mathbf{v2}-\mathbf{v0}, \quad \mathbf{pvec} = \mathbf{dir} \times \mathbf{e2}$$
    $$det = \mathbf{e1} \cdot \mathbf{pvec}$$
    If $|det| < 10^{-8}$, the ray is parallel [S2, unofficial].
    $$u = (\mathbf{tvec} \cdot \mathbf{pvec})/det, \quad v = (\mathbf{dir} \cdot \mathbf{qvec})/det$$
    Hit if $0 \le u,v$ and $u+v \le 1$ [S2, unofficial].
*   **Projection Vector:**
    $$\mathbf{proj} = \mathbf{ba} \cdot \text{clamp}\left(\frac{\mathbf{ba} \cdot \mathbf{pa}}{\mathbf{ba} \cdot \mathbf{ba}}, 0, 1\right) - \mathbf{pa}$$
    Used to compute the closest point on an edge or vertex region [S2, unofficial].

**Parameters:**
- `res`: Texture/Grid resolution (default 50) [S2, unofficial].
- `is_closed`: Boolean flag indicating if the mesh is closed, required for topological sign determination [S4].
- `trunc_dist`: Truncation distance for open meshes (skirt length) [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def point_triangle_distance(p, a, b, c):
    """
    Computes the squared distance from point p to triangle abc.
    Returns (dist_sq, closest_point).
    """
    ab = b - a
    ac = c - a
    ap = p - a

    d1 = np.dot(ab, ap)
    d2 = np.dot(ac, ap)
    if d1 <= 0.0 and d2 <= 0.0:
        return np.dot(ap, ap), a

    bp = p - b
    d3 = np.dot(ab, bp)
    d4 = np.dot(ac, bp)
    if d3 >= 0.0 and d4 <= d3:
        vc = d1 * d4 - d3 * d2
        if vc <= 0.0 and d1 >= 0.0 and d3 <= 0.0:
            v = d1 / (d1 - d3)
            return np.dot(bp + v * ab, bp + v * ab), b + v * ab

    cp = p - c
    d5 = np.dot(ab, cp)
    d6 = np.dot(ac, cp)
    if d6 >= 0.0 and d5 <= d6:
        vb = d5 * d2 - d6 * d1
        if vb <= 0.0 and d1 >= 0.0 and d6 <= 0.0:
            w = d1 / (d1 - d5)
            return np.dot(cp + w * ac, cp + w * ac), c + w * ac

    denom = 1.0 / (d1 * d4 - d2 * d3)
    v = (d2 * d4 - d3 * d5) * denom
    w = (d1 * d5 - d2 * d3) * denom
    closest = a + ab * v + ac * w
    return np.dot(p - closest, p - closest), closest

def mesh_to_sdf(mesh_vertices, mesh_faces, grid_resolution=50, is_closed=True):
    """
    Generates a Signed Distance Field from a mesh.
    mesh_vertices: Nx3 array
    mesh_faces: Mx3 array of indices
    grid_resolution: int
    is_closed: bool
    """
    grid = np.zeros((grid_resolution, grid_resolution, grid_resolution))
    
    tri_a = mesh_vertices[mesh_faces[:, 0]]
    tri_b = mesh_vertices[mesh_faces[:, 1]]
    tri_c = mesh_vertices[mesh_faces[:, 2]]
    
    for x in range(grid_resolution):
        for y in range(grid_resolution):
            for z in range(grid_resolution):
                pos = np.array([x, y, z]) / (grid_resolution - 1) * 2.0 - 1.0
                
                min_dist_sq = float('inf')
                closest_point = None
                closest_normal = None
                
                for i in range(len(mesh_faces)):
                    a = tri_a[i]
                    b = tri_b[i]
                    c = tri_c[i]
                    
                    dist_sq, cp = point_triangle_distance(pos, a, b, c)
                    
                    if dist_sq < min_dist_sq:
                        min_dist_sq = dist_sq
                        closest_point = cp
                        normal = np.cross(b - a, c - a)
                        norm_len = np.linalg.norm(normal)
                        if norm_len > 1e-8:
                            normal = normal / norm_len
                        closest_normal = normal
                
                dist = math.sqrt(min_dist_sq)
                
                sign = 1.0
                if is_closed and closest_normal is not None:
                    vec_to_p = pos - closest_point
                    if np.dot(vec_to_p, closest_normal) < 0:
                        sign = -1.0
                
                grid[x, y, z] = sign * dist
                
    return grid
```

**Edge Cases and Pitfalls:**
- **Open Meshes:** Open meshes grow a "skirt" up to `trunc_dist` past their rim [S3, unofficial].
- **Winding Order:** The mesh winding must be counter-clockwise seen from outside (normal defined by `cross(v1 - v0, v2 - v0)` pointing out) [S3, unofficial].
- **Sharp Corners:** Edges/corners with faces meeting at under 90 degrees may take the wrong side due to nearest-face tie-breaking by lowest index [S3, unofficial].
- **Precision/Implementation Pitfall:** Some GPU implementations calculate intersection hits but output `abs(dist)` regardless of inside/outside status, effectively producing an unsigned field [S2, unofficial].

## Subtopics

### [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md)
Converts explicit geometry (triangles or oriented point clouds) into a Truncated SDF voxel grid by computing closest-surface distance and assigning sign based on normals. It differs from [Projective TSDF Fusion](3649-mesh-to-tsdf-sequence-encoding-pipeline.md) by operating directly on geometric primitives rather than depth images, making it suitable for upstream reconstruction pipelines where calibrated camera poses are unavailable.

### [SDF Dataset Normalization](3787-sdf-dataset-normalization.md)
Preprocesses a 3D mesh's vertex coordinates to fit within a standardized canonical space, typically a unit cube or sphere centered at the origin. It solves inconsistent scale and position in input meshes, which is critical for downstream volume rendering pipelines where grid resolution and sampling bounds depend on predictable input dimensions. It differs from [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md) and [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md) by operating on the source mesh geometry directly before discretization.

### [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md)
Converts a triangle mesh into a sparse, tree-structured signed or unsigned distance grid using OpenVDB, storing only active voxels near the surface for memory efficiency. It differs from dense voxelization by dynamically allocating memory only for the narrow band around the mesh, making it suitable for large or complex models where ray-marched rendering or fast implicit queries are required.

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$ for brute-force voxelization, where $N$ is the number of triangles and $V$ is the number of voxels [S2, unofficial]. Using spatial acceleration structures (AABB trees) reduces query time to $O(\log N)$ per voxel [S4].
- **Memory Complexity:** $O(V)$ for dense grids [S2, unofficial]. Sparse structures reduce this to $O(N)$ near the surface [S1, unofficial].
- **Performance:** Depends heavily on mesh complexity and grid resolution. GPU implementations can parallelize triangle iteration [S2, unofficial].
- **Strengths:** Robust rendering of complex CAD meshes; supports CSG operations; differentiable depth rendering via implicit function theorem [S1, unofficial].
- **Limitations:** Open meshes require special handling (skirt); sharp angles can cause sign errors; brute-force is slow for high-poly meshes [S3, unofficial].

## Background
The method originates from the need to render explicit triangle meshes (e.g., STL files) using implicit surface rendering techniques, such as interval-arithmetic bisection [S1, unofficial]. It is implemented in libraries like `isoray` for robust iso-surface rendering and CGAL's `Isosurfacing_3` module for contouring mesh offsets [S1, unofficial][S4]. The approach allows CAD meshes to be rendered with the same robustness as analytic implicit functions [S1, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
[S4] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S5] here, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
