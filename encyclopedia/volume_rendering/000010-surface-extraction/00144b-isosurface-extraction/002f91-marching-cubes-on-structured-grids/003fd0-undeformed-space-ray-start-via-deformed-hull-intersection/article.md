# Undeformed Space Ray Start via Deformed Hull Intersection

Also called: Approximate Inverse via Hull, Dg^-1 approximation

## Overview
This method finds the ray start point $\hat{x}_0 = D^{-1}(p)$ in undeformed space without computing the inverse deformation map $D^{-1}$ directly [S1]. It solves the problem where $D$ may not be invertible or well-defined at the camera position by intersecting the deformed view ray with a deformed explicit hull mesh that contains the surface [S1].

## How it works
**Inputs:**
*   Deformation function $D(x)$ mapping undeformed space $x$ to deformed space $p$ [S1].
*   View ray in deformed space [S1].

**Outputs:**
*   Approximate undeformed ray start point $\hat{x}_0$ [S1].

**Data Structures:**
*   **Hull Mesh:** A triangle mesh defined in undeformed space with 3D vertex coordinates and triangle connectivity indices [S1].
*   The mesh must completely contain the target isosurface [S1].

**Steps:**
1.  **Generate Undeformed Hull:** Create a low-resolution explicit hull in undeformed space that completely contains the isosurface [S1]. The default method is Marching Cubes because it is easy to implement efficiently on the GPU [S1]. To ensure the hull contains the surface, offset the original isosurface by the marching cubes diagonal cell size before generation [S1]. This is valid if the surface is defined by an SDF, as offsets correspond directly to distances [S1].
2.  **Deform Hull:** Apply the forward deformation transform $D$ to all vertices of the hull mesh to create a deformed hull in deformed space [S1].
3.  **Intersect Ray with Deformed Hull:** Find the intersection point $p$ between the deformed space view ray and the deformed hull mesh [S1]. Implementation can use rasterization of the hull mesh to find the primary intersection point, or explicit ray-triangle intersections [S1].
4.  **Compute Approximate Inverse:** Obtain the corresponding object-space location $\hat{x}$ using the barycentric coordinates of the intersected triangle [S1]. This is performed automatically by hardware: object-space vertex positions are interpolated from vertex to fragment shading units by the rasterizer [S1]. The result is the approximate inverse $Dg^{-1}(p) \approx D^{-1}(p)$ [S1].

**Formula:**
The approximate inverse is constructed as:
$$ Dg^{-1}(p) \approx D^{-1}(p) $$
where $Dg^{-1}$ is the inverse derived from the deformed hull's linear interpolation [S1].

**Parameters:**
*   **Hull Generation Method:** Marching Cubes is used by default because it is easy to implement efficiently on the GPU [S1]. Alternatives like Wyvill et al. 1986 were not explored [S1].
*   **Offset Distance:** The marching cubes diagonal cell size [S1].

**Pitfalls:**
*   **Non-Conservative Extraction:** Standard Marching Cubes is not conservative [S1]. If the hull does not completely contain the isosurface, parts of the surface outside the hull will not be rendered, and hull faces cutting through the isosurface will reveal the inside of the object [S1].
*   **Precision:** The method introduces error from the linearization of the inverse via hull interpolation and numerical integration of the trajectory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_undeformed_ray_start(view_ray_origin, view_ray_direction, deformation_func, sdf_field, cell_diagonal_size):
    # Step 1: Generate Undeformed Hull
    # Offset the SDF to ensure the hull completely contains the original isosurface.
    # For a standard SDF (inside < 0, outside > 0), extracting at -offset expands the mesh [G].
    offset_value = -cell_diagonal_size
    hull_undeformed = marching_cubes_impl(sdf_field, isovalue=offset_value)

    # Step 2: Deform Hull
    hull_deformed = TriangleMesh()
    for v in hull_undeformed.vertices:
        hull_deformed.vertices.append(deformation_func(v))
    hull_deformed.triangles = hull_undeformed.triangles

    # Step 3: Intersect Ray with Deformed Hull
    # Find primary intersection point p in deformed space.
    # Implementation uses rasterization or explicit ray-triangle intersections [S1].
    hit = ray_triangle_intersect(view_ray_origin, view_ray_direction, hull_deformed)

    if hit is None:
        return None

    p = hit.point
    tri_idx = hit.triangle_index
    bary = hit.barycentric

    # Step 4: Compute Approximate Inverse
    # Use barycentric coordinates to interpolate undeformed vertex positions.
    # This is performed automatically by hardware rasterization [S1].
    v0 = hull_undeformed.vertices[hull_deformed.triangles[tri_idx][0]]
    v1 = hull_undeformed.vertices[hull_deformed.triangles[tri_idx][1]]
    v2 = hull_undeformed.vertices[hull_deformed.triangles[tri_idx][2]]

    x_hat = bary[0] * v0 + bary[1] * v1 + bary[2] * v2
    return x_hat

class TriangleMesh:
    def __init__(self):
        self.vertices = []
        self.triangles = []

def marching_cubes_impl(sdf_field, isovalue):
    # Sources do not provide the Marching Cubes implementation details [S1].
    # Standard algorithm generates a triangle mesh from a scalar field at the given isovalue.
    mesh = TriangleMesh()
    # ... standard MC logic omitted for brevity, returns mesh containing surface ...
    return mesh

def ray_triangle_intersect(origin, direction, mesh):
    # Standard ray-triangle intersection (Möller–Trumbore algorithm) [G].
    closest_t = float('inf')
    closest_hit = None
    for i, tri in enumerate(mesh.triangles):
        v0 = mesh.vertices[tri[0]]
        v1 = mesh.vertices[tri[1]]
        v2 = mesh.vertices[tri[2]]
        edge1 = v1 - v0
        edge2 = v2 - v0
        h = cross(direction, edge2)
        a = dot(edge1, h)
        if a > -1e-8 and a < 1e-8:
            continue
        f = 1.0 / a
        s = origin - v0
        u = f * dot(s, h)
        if u < 0.0 or u > 1.0:
            continue
        q = cross(s, edge1)
        v = f * dot(direction, q)
        if v < 0.0 or u + v > 1.0:
            continue
        t = f * dot(edge2, q)
        if t > 1e-8 and t < closest_t:
            closest_t = t
            closest_hit = {'point': origin + t * direction, 'triangle_index': i, 'barycentric': (1.0 - u - v, u, v)}
    return closest_hit
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires storing the undeformed and deformed hull meshes. Not given in the sources.
*   **Strengths:** Avoids direct inversion of $D$, which may not exist or be well-defined. Efficient on GPU due to rasterization support [S1].
*   **Limitations:** Approximate; introduces error from linearization. Requires the hull to completely contain the surface, necessitating an offset [S1].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
This method was introduced in the context of non-linear sphere tracing for rendering deformed signed distance fields [S1]. It generalizes the bounding-box technique for determining an initial value proposed by Barr [1986] [S1]. The approach leverages the efficiency of GPU rasterization to approximate the inverse deformation map, allowing for robust ray tracing in deformed spaces without explicit inverse functions [S1].

## Sources
[S1] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
