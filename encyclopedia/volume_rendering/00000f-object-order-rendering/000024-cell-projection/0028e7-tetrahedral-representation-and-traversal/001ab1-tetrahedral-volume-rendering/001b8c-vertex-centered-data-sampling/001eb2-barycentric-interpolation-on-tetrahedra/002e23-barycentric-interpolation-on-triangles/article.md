# Barycentric Interpolation on Triangles

Also called: Barycentric Interpolation of Mesh Attributes, smoothColor, smoothNormal, smoothUV

## Overview
Barycentric interpolation on triangles is the method for computing vertex attributes (such as colors, normals, or UV coordinates) at an arbitrary point within a triangle, given the vertex attributes and the point's barycentric coordinates. It serves as the fundamental sampling primitive for triangle-based rendering and mesh processing, providing linearly interpolated values that vary smoothly across the triangle surface. It is the 2D simplex counterpart to [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md), which extends this concept to 3D volumes using four vertices.

## How it works
**Inputs:**
- Triangle vertex indices $i_0, i_1, i_2$ from the mesh geometry.
- Barycentric coordinates $u, v$ of the intersection point relative to the triangle vertices.
- Vertex attribute arrays (e.g., `colors`, `normals`, `uv`) indexed by vertex ID.

**Outputs:**
- Interpolated attribute value $A$ at the hit point.

**Data Structures:**
- The triangle is defined by three vertices $V_0, V_1, V_2$.
- Attributes are stored per-vertex. For a triangle, we access `attr[i_0]`, `attr[i_1]`, and `attr[i_2]`.

**Steps:**
1. Retrieve the triangle vertex indices $i_0, i_1, i_2$ using the primitive ID from the ray hit.
2. Extract the barycentric coordinates $u$ and $v$ from the hit record.
3. Compute the third barycentric coordinate $w$:
   $$ w = 1 - u - v $$
4. Clamp $w$ to the range $[0.0, 1.0]$ to handle floating-point precision errors or boundary cases where $w$ might slightly exceed valid bounds due to numerical instability [S1, unofficial].
5. Perform the linear interpolation:
   $$ A = w \cdot A_0 + u \cdot A_1 + v \cdot A_2 $$
   where $A_0, A_1, A_2$ are the attributes at vertices $i_0, i_1, i_2$ respectively.
6. If the attribute is a normal vector, normalize the result:
   $$ \hat{N} = \frac{w \cdot N_0 + u \cdot N_1 + v \cdot N_2}{\|w \cdot N_0 + u \cdot N_1 + v \cdot N_2\|} $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barycentric_interpolate_triangle(tri_indices, u, v, attr_arrays, attr_type='color'):
    """
    Interpolate an attribute over a triangle using barycentric coordinates.
    
    Args:
        tri_indices: Tuple of 3 vertex indices (i0, i1, i2).
        u: Barycentric coordinate u.
        v: Barycentric coordinate v.
        attr_arrays: Dictionary mapping attribute names to vertex attribute arrays.
        attr_type: String indicating the type of attribute ('color', 'normal', 'uv').
        
    Returns:
        Interpolated attribute value.
    """
    i0, i1, i2 = tri_indices
    
    # Compute w and clamp to handle precision errors [S1, unofficial]
    w = 1.0 - u - v
    w = max(0.0, min(1.0, w))
    
    # Check if attribute exists [S1, unofficial]
    if attr_type not in attr_arrays:
        # Log error or return undefined behavior depending on caller [S1, unofficial]
        return None
        
    attr0 = attr_arrays[attr_type][i0]
    attr1 = attr_arrays[attr_type][i1]
    attr2 = attr_arrays[attr_type][i2]
    
    # Linear interpolation
    result = w * attr0 + u * attr1 + v * attr2
    
    # Normalize if normal
    if attr_type == 'normal':
        result = result / (np.linalg.norm(result) + 1e-8) # [G] small epsilon to avoid div by zero
        
    return result
```

**Parameters:**
- `hit.barycentricCoord().u` and `.v`: The barycentric coordinates of the intersection point [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision Errors:** The sum $u + v + w$ should be 1.0, but floating-point arithmetic may cause slight deviations. Clamping $w$ to $[0, 1]$ prevents out-of-bounds access or invalid weights [S1, unofficial].
- **Missing Attributes:** If the mesh does not contain the requested attribute (e.g., normals or UVs), the function must handle this gracefully, typically by logging an error or returning a default value [S1, unofficial].
- **Degenerate Triangles:** If the triangle vertices are collinear or identical, the barycentric coordinates may be undefined or unstable. This is generally handled at the ray-triangle intersection stage.

## Operating characteristics
- **Time Complexity:** $O(1)$ per triangle hit. The computation involves a few arithmetic operations and array lookups.
- **Memory Complexity:** $O(1)$ auxiliary space. The attribute data is accessed directly from vertex buffers.
- **Strengths:** Simple, fast, and provides smooth interpolation across triangle surfaces. Essential for shading and texturing.
- **Limitations:** Linear interpolation does not account for curvature, leading to faceted appearance for normals on coarse meshes. Requires pre-computed vertex attributes.

## Background
Barycentric interpolation is a fundamental technique in computer graphics, dating back to the early days of rasterization and ray tracing. It is widely used in triangle-based rendering pipelines for shading, texturing, and normal interpolation. The method is standard in graphics APIs and rendering engines.

## Sources
[S1] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
[S2] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
