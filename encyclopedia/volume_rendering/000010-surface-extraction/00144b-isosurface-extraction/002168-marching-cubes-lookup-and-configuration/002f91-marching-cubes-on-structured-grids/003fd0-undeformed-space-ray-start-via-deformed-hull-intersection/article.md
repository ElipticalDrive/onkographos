# Undeformed Space Ray Start via Deformed Hull Intersection

Also called: Approximate Inverse via Hull, Dg^-1 approximation

## Overview
This method determines the starting point $\hat{x}_0$ in undeformed (object) space for a ray tracing trajectory in a deformed volume, avoiding the direct computation of the inverse deformation map $D^{-1}$. It solves the problem where $D$ may not be invertible or well-defined at the camera position by intersecting the deformed view ray with a deformed explicit hull mesh that contains the surface. It differs from [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md) and [Position Error Minimized Marching Cubes](7435-position-error-minimized-marching-cubes.md) by prioritizing the robust generation of a bounding hull for inverse approximation rather than optimizing the isosurface extraction itself.

## How it works
**Inputs:**
*   Deformation function $D(x)$ mapping undeformed space $x$ to deformed space $p$.
*   Isosurface definition (typically a Signed Distance Field, SDF).
*   View ray in deformed space.

**Outputs:**
*   Approximate undeformed ray start point $\hat{x}_0$.

**Data Structures:**
*   **Hull Mesh:** A triangle mesh defined in undeformed space.
    *   Vertices: 3D coordinates $(x, y, z)$.
    *   Indices: Triangle connectivity.
    *   The mesh must completely contain the target isosurface.

**Steps:**
1.  **Generate Undeformed Hull:** Create a low-resolution explicit hull in undeformed space that completely contains the isosurface.
    *   The default method is Marching Cubes [S1].
    *   To ensure the hull contains the surface, offset the original isosurface by the marching cubes diagonal cell size before generation. This is valid if the surface is defined by an SDF, as offsets correspond directly to distances [S1].
2.  **Deform Hull:** Apply the forward deformation transform $D$ to all vertices of the hull mesh to create a deformed hull in deformed space.
3.  **Intersect Ray with Deformed Hull:** Find the intersection point $p$ between the deformed space view ray and the deformed hull mesh.
    *   Implementation can use rasterization of the hull mesh to find the primary intersection point, or explicit ray-triangle intersections [S1].
4.  **Compute Approximate Inverse:** Obtain the corresponding object-space location $\hat{x}$ using the barycentric coordinates of the intersected triangle.
    *   This is performed automatically by hardware: object-space vertex positions are interpolated from vertex to fragment shading units by the rasterizer [S1].
    *   The result is the approximate inverse $Dg^{-1}(p) \approx D^{-1}(p)$ [S1].

**Formula:**
The approximate inverse is constructed as:
$$ Dg^{-1}(p) \approx D^{-1}(p) $$
where $Dg^{-1}$ is the inverse derived from the deformed hull's linear interpolation [S1].

**Parameters:**
*   **Hull Generation Method:** Marching Cubes is used by default because it is easy to implement efficiently on the GPU. Alternatives like Wyvill et al. 1986 were not explored [S1].
*   **Offset Distance:** The marching cubes diagonal cell size [S1].

**Pitfalls:**
*   **Non-Conservative Extraction:** Standard Marching Cubes is not conservative. If the hull does not completely contain the isosurface, parts of the surface outside the hull will not be rendered, and hull faces cutting through the isosurface will reveal the inside of the object [S1].
*   **Precision:** The method introduces error from the linearization of the inverse via hull interpolation and numerical integration of the trajectory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_undeformed_ray_start(view_ray_deformed, deformation_func, sdf_field, cell_diagonal_size):
    """
    Computes the approximate undeformed ray start point using a deformed hull.
    
    Args:
        view_ray_deformed: Ray origin and direction in deformed space.
        deformation_func: Function D(x) mapping undeformed to deformed space.
        sdf_field: Signed Distance Field defining the isosurface.
        cell_diagonal_size: The diagonal size of the marching cubes cells.
        
    Returns:
        x_hat: Approximate undeformed ray start point.
    """
    # Step 1: Generate Undeformed Hull
    # Offset the SDF to ensure the hull contains the surface
    # Note: This assumes SDF offsets correspond directly to distances [S1]
    offset_sdf = sdf_field + cell_diagonal_size
    
    # Generate low-resolution hull using Marching Cubes on the offset SDF
    # This produces a mesh in undeformed space
    hull_mesh_undeformed = marching_cubes(offset_sdf, resolution="low")
    
    # Step 2: Deform Hull
    # Apply forward transform D to all vertices
    hull_mesh_deformed = HullMesh()
    for vertex in hull_mesh_undeformed.vertices:
        deformed_vertex = deformation_func(vertex)
        hull_mesh_deformed.add_vertex(deformed_vertex)
    hull_mesh_deformed.copy_connectivity(hull_mesh_undeformed)
    
    # Step 3: Intersect Ray with Deformed Hull
    # Find primary intersection point p in deformed space
    # Implementation can use rasterization or ray-triangle intersection [S1]
    intersection = ray_intersect_mesh(view_ray_deformed, hull_mesh_deformed)
    
    if intersection is None:
        # Handle case where ray misses hull (e.g., return infinity or error)
        return None
        
    p = intersection.point
    triangle = intersection.triangle
    
    # Step 4: Compute Approximate Inverse
    # Use barycentric coordinates to interpolate undeformed vertex positions
    # This is done automatically by hardware rasterization [S1]
    # Retrieve undeformed vertices of the intersected triangle
    v0_undeformed = hull_mesh_undeformed.vertices[triangle.v0]
    v1_undeformed = hull_mesh_undeformed.vertices[triangle.v1]
    v2_undeformed = hull_mesh_undeformed.vertices[triangle.v2]
    
    # Interpolate using barycentric coordinates (b0, b1, b2) from intersection
    x_hat = (b0 * v0_undeformed) + (b1 * v1_undeformed) + (b2 * v2_undeformed)
    
    return x_hat
```

## Operating characteristics
*   **Time Complexity:** Depends on the hull generation and intersection cost. Hull generation is $O(N)$ where $N$ is the number of voxels in the low-resolution grid. Intersection is $O(M)$ where $M$ is the number of triangles in the hull, or $O(1)$ with spatial acceleration structures.
*   **Memory Complexity:** Requires storing the undeformed and deformed hull meshes. The resolution is low, so memory overhead is small compared to the full volume [S1].
*   **Strengths:** Avoids direct inversion of $D$, which may not exist or be well-defined. Efficient on GPU due to rasterization support [S1].
*   **Limitations:** Approximate; introduces error from linearization. Requires the hull to completely contain the surface, necessitating an offset [S1].
*   **When to use:** When rendering deformed volumes where the inverse deformation is difficult to compute directly.
*   **When not to use:** When high precision is required for the ray start point, or when the deformation is simple enough to invert analytically.

## Background
This method was introduced in the context of non-linear sphere tracing for rendering deformed signed distance fields. It generalizes the bounding-box technique for determining an initial value proposed by Barr [1986] [S1]. The approach leverages the efficiency of GPU rasterization to approximate the inverse deformation map, allowing for robust ray tracing in deformed spaces without explicit inverse functions.

## Sources
[S1] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
