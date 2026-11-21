# Barycentric Interpolation on Triangles

Also called: Barycentric Interpolation of Mesh Attributes, smoothColor, smoothNormal, smoothUV

## Overview
Barycentric interpolation on triangles evaluates vertex attributes (normals, colors, UVs) at a ray hit point using barycentric coordinates [S2, unofficial]. It provides a linearly varying field over the 2D simplex defined by the triangle. Unlike [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md), which operates in 3D volume using four vertices, this method operates on the 2D surface using three vertices. It is distinct from [Clough–Tocher Bézier Patch Interpolation](9134-clough-tocher-b-zier-patch-interpolation.md) by being strictly linear and C0 continuous, which is computationally cheaper but prone to visible shading discontinuities across edges if vertex normals are not shared [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A triangle defined by three vertices $V_0, V_1, V_2$, each possessing an attribute value $A_0, A_1, A_2$ (e.g., color vector, normal vector, UV pair) [S2, unofficial]. The intersection point $P$ on the triangle, represented by barycentric coordinates $u$ and $v$ [S1, unofficial].
- **Output:** The interpolated attribute value $A$ at point $P$ [S2, unofficial].

**Data Structures:**
- **Triangle Indices:** An array or structure storing the vertex indices for each primitive. Access is via `mesh.triangles()[hit.primitive().triID]` [S1, unofficial][S2, unofficial].
- **Barycentric Coordinates:** Two floating-point values $u$ and $v$ provided by the ray-triangle intersection routine. The third coordinate $w$ is derived [S1, unofficial][S2, unofficial].
- **Vertex Attributes:** A lookup table or structure mapping vertex indices to attribute vectors. Dimension depends on attribute (e.g., 3 for RGB/Normal, 2 for UV) [G].

**Steps:**
1. Retrieve the three vertex indices of the hit triangle from the mesh data structure using the primitive ID from the hit record [S1, unofficial][S2, unofficial].
2. Extract the barycentric coordinates $u$ and $v$ from the hit record [S1, unofficial][S2, unofficial].
3. Compute the third barycentric coordinate $w$ using the constraint that the sum of barycentric coordinates is 1:
   $$w = 1 - u - v$$
4. Clamp $w$ to the range $[0.0, 1.0]$ to handle floating-point precision errors or boundary cases where $w$ might slightly exceed valid bounds due to numerical instability [S1, unofficial].
5. Perform the weighted sum of the vertex attributes:
   $$A = w \cdot A_0 + u \cdot A_1 + v \cdot A_2$$
6. If the attribute is a normal vector, normalize the result to ensure it remains a unit vector:
   $$\hat{N} = \frac{A}{\|A\|}$$
   [S2, unofficial]. This normalization is required because linear interpolation of normals does not preserve unit length [S2, unofficial].
7. If the mesh lacks the requested attribute, handle the error gracefully (e.g., return a default value or log an error) [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barycentric_interpolate(triangle_indices, vertex_attributes, u, v, attribute_type='color'):
    i0, i1, i2 = triangle_indices
    w = 1.0 - u - v
    w = max(0.0, min(1.0, w))
    A0 = vertex_attributes[i0]
    A1 = vertex_attributes[i1]
    A2 = vertex_attributes[i2]
    A = [w*A0[0] + u*A1[0] + v*A2[0], w*A0[1] + u*A1[1] + v*A2[1], w*A0[2] + u*A1[2] + v*A2[2]]
    if attribute_type == 'normal':
        norm = (A[0]**2 + A[1]**2 + A[2]**2)**0.5
        if norm == 0:
            return [0.0, 1.0, 0.0]
        A = [A[0]/norm, A[1]/norm, A[2]/norm]
    return A
```

**Parameters:**
- $u, v$: Barycentric coordinates of the intersection point [S1, unofficial]. Typically in $[0, 1]$.
- $A_0, A_1, A_2$: Vertex attributes. Dimension depends on attribute (e.g., 3 for RGB/Normal, 2 for UV) [G].

**Edge Cases and Pitfalls:**
- **Precision Errors:** Floating-point arithmetic may result in $u + v + w \neq 1$ or $w < 0$ slightly. Clamping $w$ to $[0, 1]$ handles these boundary cases [S1, unofficial].
- **Missing Attributes:** If the mesh does not have normals, colors, or UVs, the function must check for existence before accessing [S1, unofficial][S2, unofficial].
- **Normal Interpolation:** Linear interpolation of normals distorts the vector length and direction relative to the surface curvature. Normalization is required [S2, unofficial].

## Subtopics

### [Clough–Tocher Bézier Patch Interpolation](9134-clough-tocher-b-zier-patch-interpolation.md)
Replaces linear barycentric interpolation with a cubic Bézier patch construction to interpolate feature values and gradients over a triangle, guaranteeing C1 continuity everywhere and C∞ within each triangle. It solves the problem of visible seam artifacts caused by gradient discontinuities across triangle edges in fully opaque meshes, which arise because standard barycentric interpolation produces a C0 feature field. Unlike the parent method, [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md), which only uses vertex values, this method requires per-vertex gradients to define the patch geometry.

### Component-wise Tensor Interpolation
A method for interpolating second-order tensor fields by treating the tensor as a linear combination of surrounding edge tensors, avoiding the orientation ambiguities and non-unit vector issues of interpolating eigenvectors directly [S3, unofficial]. It calculates barycentric coordinates $\beta_k$ for the point $p$ and computes the interpolated tensor component $T_{ij}$ as:
$$T_{ij} = \frac{\sum_{0 < k < 3} \beta_k T_{ij}^k}{\sum_{0 < k < 3} \beta_k}$$
[S3, unofficial]. Pitfalls include naive interpolation of eigenvectors with different orientations causing rotation of vectors and potential zero vectors due to numerical instabilities [S3, unofficial]. This method preserves symmetry and reduces computational effort compared to interpolating raw MRI data or derived quantities like eigenvectors [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per triangle hit. The computation involves a few floating-point subtractions, multiplications, and additions [G].
- **Memory Complexity:** $O(1)$ auxiliary memory. Requires storage for the vertex attributes and the barycentric coordinates [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Simple to implement, computationally inexpensive, works for any scalar or vector attribute [G].
- **Limitations:** Produces C0 continuous fields, leading to visible shading discontinuities (faceting) on curved surfaces if vertex normals are not shared or if the mesh is coarse [G].
- **When to use:** Real-time rendering, simple mesh visualization, UV mapping, color interpolation [G].
- **When not to use:** High-quality offline rendering requiring smooth shading, tensor field visualization requiring C1 continuity, or when gradient information is available and needed [G].

## Background
Barycentric interpolation on triangles is a fundamental technique in computer graphics, originating from the mathematical theory of barycentric coordinates in geometry [G]. It has been used since the early days of rasterization and ray tracing to interpolate vertex attributes across triangle surfaces [G]. The method is standard in graphics APIs (OpenGL, DirectX) and rendering engines [G]. The specific implementation details regarding clamping and normalization are documented in modern ray tracing and Gaussian Splatting frameworks (e.g., 3DGS-FLoD, whu-lyh/flod) [S1, unofficial][S2, unofficial]. The extension to tensor fields via component-wise interpolation is discussed in the context of diffusion tensor imaging visualization [S3, unofficial].

## Sources
[S1] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
[S2] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
[S3] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
