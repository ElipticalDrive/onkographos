# Signed Distance Field from Mesh

Also called: Signed Mesh SDF, Triangle Mesh SDF, Mesh SDF Rendering

## Overview
This method converts a triangulated surface mesh into a 3D Signed Distance Field (SDF) texture or implicit function, where the scalar value at any point represents the signed Euclidean distance to the nearest surface triangle [S1, S2, unofficial][S4]. The sign is negative inside the mesh and positive outside, determined by the closest triangle's face normal or a topological side-of-mesh test [S3, unofficial][S4]. It differs from [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md) by evaluating the distance field directly from the explicit mesh geometry (via raycasting or spatial acceleration structures) rather than projecting 2D slices, allowing for exact analytical distances without rasterization artifacts [S2, unofficial][S4].

## How it works
**Inputs and Outputs:**
*   **Input:** A closed triangulated mesh (e.g., STL format) with consistent winding (counter-clockwise when viewed from outside) [S1, S3, unofficial].
*   **Output:** An implicit function $f(\mathbf{p})$ returning the signed distance, or a 3D texture where each voxel stores the distance value [S1, S2, unofficial].

**Data Structures:**
1.  **Mesh Storage:** Vertices and triangle indices. In GPU implementations, vertices may be stored in a texture (`meshtex`) with 3 floats (RGB) per texel and a stride of 16384 texels per row [S2, unofficial].
2.  **Spatial Acceleration:** An Axis-Aligned Bounding Box (AABB) tree built from the mesh faces to accelerate closest-point queries [S4].
3.  **Side-of-Mesh Functor:** A structure (e.g., `Side_of_triangle_mesh`) that determines if a query point is on the bounded (inside) or unbounded (outside) side of the mesh [S4].

**Algorithm Steps:**
The method generally follows one of two approaches: direct per-voxel raycasting (for GPU textures) or spatial query (for CPU/implicit functions).

### Approach A: GPU Raycasting (Per-Voxel)
1.  **Initialize:** For each voxel in the output grid (resolution `res` x `res` x `res`), calculate the voxel center $\mathbf{pos}$ in world space, typically mapped to $[-1, 1]^3$ via $\mathbf{pos} = (\mathbf{uv} / \text{res}) \times 2.0 - 1.0$ [S2, unofficial].
2.  **Distance Search:** Initialize `dist` to a large value (e.g., 10000.0) [S2, unofficial].
3.  **Triangle Iteration:** Loop through all triangles in the mesh. For each triangle with vertices $\mathbf{a}, \mathbf{b}, \mathbf{c}$:
    *   Compute the Euclidean distance $d$ from $\mathbf{pos}$ to the triangle using a closest-point function `udTriangle` [S2, unofficial].
    *   Update `dist = min(dist, d)` [S2, unofficial].
4.  **Sign Determination (Optional/Implicit):** In some implementations, the sign is ignored or handled by taking the absolute value `abs(dist)` if the winding/sign logic is not fully integrated into the distance kernel [S2, unofficial]. In robust implementations, the sign is determined by the closest triangle's normal [S3, unofficial].
5.  **Output:** Store the final distance in the output texture (e.g., `fragColor.r`) [S2, unofficial].

### Approach B: Spatial Query (CPU/Implicit)
1.  **Build Tree:** Construct an `AABB_tree` from the mesh faces [S4].
2.  **Initialize Oracle:** Create an `Offset_oracle` struct containing the `is_closed` flag, the `AABB_tree`, and the `Side_of_triangle_mesh` functor [S4].
3.  **Query Point:** For a query point $\mathbf{p}$:
    *   Find the closest point $\mathbf{cp}$ on the mesh surface using `tree.closest_point(p)` [S4].
    *   Compute the unsigned distance $d = \sqrt{(\mathbf{p} - \mathbf{cp})^2}$ [S4].
    *   Determine the sign: If `is_closed` is true and `sotm(p)` returns `ON_BOUNDED_SIDE`, negate the distance: $d = -d$ [S4].
4.  **Return:** Return $d$ [S4].

**Formulas:**

*   **Triangle Distance (`udTriangle`):**
    Let $\mathbf{pa} = \mathbf{p} - \mathbf{a}$ and $\mathbf{ba} = \mathbf{b} - \mathbf{a}$.
    The projection of $\mathbf{pa}$ onto $\mathbf{ba}$ is clamped:
    $$ \mathbf{proj} = \mathbf{ba} \cdot \text{clamp}\left(\frac{\mathbf{ba} \cdot \mathbf{pa}}{\mathbf{ba} \cdot \mathbf{ba}}, 0, 1\right) - \mathbf{pa} $$
    The distance is the minimum of the distance to the triangle plane (if outside) or the edge/vertex distances [S2, unofficial].
    If outside: $d = \sqrt{\min(\dots)}$ [S2, unofficial].
    If inside: $d = \frac{(\mathbf{n} \cdot \mathbf{pa})^2}{\mathbf{n} \cdot \mathbf{n}}$ where $\mathbf{n}$ is the triangle normal [S2, unofficial].
    Inside check for winding: $\sum \text{sign}(\mathbf{ba} \times \mathbf{n} \cdot \mathbf{pa}) < 2$ [S2, unofficial].

*   **Ray-Triangle Intersection (Möller-Trumbore):**
    Used to determine if a ray from the voxel center intersects the triangle for inside/outside counting or closest point refinement [S2, unofficial].
    $$ \mathbf{e1} = \mathbf{v1} - \mathbf{v0}, \quad \mathbf{e2} = \mathbf{v2} - \mathbf{v0} $$
    $$ \mathbf{pvec} = \mathbf{dir} \times \mathbf{e2}, \quad det = \mathbf{e1} \cdot \mathbf{pvec} $$
    If $|det| < 10^{-8}$, the ray is parallel.
    $$ u = \frac{\mathbf{tvec} \cdot \mathbf{pvec}}{det}, \quad v = \frac{\mathbf{dir} \cdot \mathbf{qvec}}{det} $$
    Hit if $0 \le u, v$ and $u + v \le 1$ [S2, unofficial].

**Parameters:**
*   `res`: Texture resolution (default 50) [S2, unofficial].
*   `is_closed`: Boolean flag indicating if the mesh is closed, required for correct sign determination in Approach B [S4].
*   `trunc_dist`: Maximum distance for skirt generation in open meshes [S3, unofficial].

**Pseudocode (Approach B - CPU/Implicit):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class AABBTree:
    """Simplified AABB tree for closest point queries."""
    def __init__(self, triangles):
        self.triangles = triangles
        # Build tree structure... [G]

    def closest_point(self, p):
        """Find closest point on any triangle to p."""
        min_dist_sq = float('inf')
        closest_pt = None
        for tri in self.triangles:
            pt = closest_point_on_triangle(p, tri)
            d_sq = squared_length(sub(p, pt))
            if d_sq < min_dist_sq:
                min_dist_sq = d_sq
                closest_pt = pt
        return closest_pt

def closest_point_on_triangle(p, tri):
    """Compute closest point on triangle to p."""
    a, b, c = tri
    ab = sub(b, a)
    ac = sub(c, a)
    ap = sub(p, a)
    
    d1 = dot(ab, ap)
    d2 = dot(ac, ap)
    if d1 <= 0.0 and d2 <= 0.0:
        return a
    
    bp = sub(p, b)
    d3 = dot(ab, bp)
    d4 = dot(ab, ab)
    if d3 >= 0.0 and d3 >= d4:
        return b
    
    vc = d1 * d4 - d3 * d2
    if vc >= 0.0 and d2 <= 0.0:
        t = d1 / d4
        return add(a, scale(ab, t))
    
    cp = sub(p, c)
    d5 = dot(ac, cp)
    d6 = dot(ac, ac)
    if d5 >= 0.0 and d5 >= d6:
        return c
    
    vb = d5 * d4 - d3 * d6
    if vb >= 0.0 and d1 <= 0.0 and d5 <= d6:
        t = vb / (d1 * d4 - d3 * d2)
        return add(a, scale(add(ab, scale(ac, t)), t))
    
    denom = 1.0 / (d1 * d4 - d3 * d2)
    t = (d2 * d4 - d3 * d6) * denom
    return add(a, scale(add(ab, scale(ac, t)), t))

def side_of_triangle_mesh(p, mesh):
    """Determine if p is inside (ON_BOUNDED_SIDE) or outside."""
    # Ray casting or winding number check [G]
    # Returns ON_BOUNDED_SIDE or ON_UNBOUNDED_SIDE
    pass

def compute_sdf(p, mesh, is_closed):
    """Compute signed distance from point p to mesh."""
    tree = AABBTree(mesh.triangles)
    cp = tree.closest_point(p)
    d = math.sqrt(squared_length(sub(p, cp)))
    
    if is_closed:
        side = side_of_triangle_mesh(p, mesh)
        if side == "ON_BOUNDED_SIDE":
            d = -d
    return d
```

## Subtopics

### [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md)
Converts explicit geometry (meshes/point clouds) to Truncated SDF values by computing closest-surface distances and assigning signs based on normals. Choose when truncation is required for integration.

### [SDF Dataset Normalization](3787-sdf-dataset-normalization.md)
Normalizes mesh vertex coordinates to the range $[-1, 1]$ for consistent SDF dataset loading. Choose when preparing data for neural networks or standardized pipelines.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$ for naive raycasting where $N$ is the number of triangles and $V$ is the number of voxels [S2, unofficial]. With an AABB tree, closest-point queries are $O(\log N)$, reducing total time to $O(V \log N)$ [S4].
*   **Memory Complexity:** $O(N)$ for mesh storage and $O(V)$ for the output SDF texture [S2, unofficial][S4].
*   **Performance:** GPU raycasting is parallelizable but limited by the number of triangles; CPU spatial queries are faster for large meshes but require building the acceleration structure [S2, unofficial][S4].
*   **Strengths:** Exact analytical distances; robust rendering with interval arithmetic [S1, unofficial]; supports CSG operations [S1, unofficial].
*   **Limitations:** Open meshes generate "skirts" up to `trunc_dist` past the rim [S3, unofficial]; inside-out meshes remain inside-out [S3, unofficial]; edges/corners with angles < 90 degrees may have incorrect signs due to nearest-face tie-breaking [S3, unofficial].
*   **When to use:** Rendering CAD meshes (e.g., brake calipers, brackets) as implicit surfaces [S1, unofficial]; generating offset meshes for isosurfacing [S4].
*   **When not to use:** Open meshes requiring precise boundary handling without skirts (use Shell mode) [S3, unofficial]; extremely high-poly meshes without spatial acceleration.

## Background
The method originates from the need to render complex CAD meshes as implicit surfaces using robust interval-arithmetic bisection tracers [S1, unofficial]. It was implemented in libraries like `isoray` to support STL files and in CGAL's `Isosurfacing_3` module for contouring mesh offsets [S4][S1, unofficial]. The technique leverages spatial acceleration structures (AABB trees) and side-of-mesh functors to efficiently compute signed distances, enabling applications like dual contouring and marching cubes on mesh-derived fields [S4].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
[S4] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
