# Vertex-centered Data Sampling

Also called: Vertex interpolation

## Overview
Vertex-centered Data Sampling reconstructs scalar field values within a tetrahedral element by interpolating the scalar data stored at the element's four vertices using the sample point's barycentric coordinates [S1, unofficial]. It is distinguished from its sibling [Cell-centered Data Sampling](974-approximate-cell-sampling.md) by relying on vertex-local data rather than averaging cell-centered values from face neighbors, thereby avoiding the complexity of locating neighbor centroids for non-coarse elements [S1, unofficial].

## How it works
**Inputs:**
*   `element_id`: Integer identifier for the current tetrahedral element.
*   `p`: The 3D sample point $(x, y, z)$ inside the tetrahedron.
*   `tet_vertices`: Array of 4 vertex positions $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ associated with `element_id`.
*   `vertex_scalars`: Array of 4 scalar values $s_0, s_1, s_2, s_3$ associated with the vertices of `element_id`.

**Outputs:**
*   `sample_value`: The interpolated scalar value at point $p$.

**Data Structures:**
*   Vertex-centered data is stored duplicated per element during the TET-AMR construction process [S1, unofficial]. This allows direct access to the four vertex scalars via the `element_id` without requiring global vertex lookups or neighbor traversal.

**Steps:**
1.  Retrieve the four vertex positions $\mathbf{v}_i$ and their corresponding scalar values $s_i$ for $i \in \{0, 1, 2, 3\}$ using `element_id` [S1, unofficial].
2.  Compute the barycentric coordinates $\lambda_0, \lambda_1, \lambda_2, \lambda_3$ of the sample point $p$ with respect to the tetrahedron defined by $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$.
3.  Calculate the interpolated value using the weighted sum:
    $$ s(p) = \sum_{i=0}^{3} \lambda_i s_i $$
4.  Return $s(p)$.

**Formulas:**
The barycentric coordinates $\lambda_i$ are defined such that:
$$ p = \sum_{i=0}^{3} \lambda_i \mathbf{v}_i $$
subject to the constraint:
$$ \sum_{i=0}^{3} \lambda_i = 1 $$
and $\lambda_i \ge 0$ for $p$ inside the tetrahedron [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_centered_sample(element_id, p, tet_vertices, vertex_scalars):
    """
    Interpolate scalar value at point p inside a tetrahedron using vertex-centered data.
    
    Args:
        element_id: ID of the tetrahedron.
        p: 3D point (x, y, z).
        tet_vertices: List of 4 vertices [(x0,y0,z0), ..., (x3,y3,z3)].
        vertex_scalars: List of 4 scalars [s0, ..., s3].
        
    Returns:
        Interpolated scalar value.
    """
    # Step 1: Retrieve data (assumed available via element_id lookup)
    # v0, v1, v2, v3 = tet_vertices
    # s0, s1, s2, s3 = vertex_scalars
    
    # Step 2: Compute barycentric coordinates
    # lambda_i are computed such that p = sum(lambda_i * v_i) and sum(lambda_i) = 1
    lambdas = compute_barycentric_coordinates(p, tet_vertices)
    
    # Step 3: Interpolate
    sample_value = 0.0
    for i in range(4):
        sample_value += lambdas[i] * vertex_scalars[i]
        
    return sample_value

def compute_barycentric_coordinates(p, vertices):
    """
    Compute barycentric coordinates of point p relative to tetrahedron vertices.
    Uses the volume ratio method.
    """
    v0, v1, v2, v3 = vertices
    
    # Volume of the tetrahedron
    # V = (1/6) * |det(v1-v0, v2-v0, v3-v0)|
    # We can use signed volume for consistent orientation
    def signed_volume(a, b, c, d):
        return dot(cross(b-a, c-a), d-a) / 6.0

    total_volume = signed_volume(v0, v1, v2, v3)
    
    # Barycentric coordinate for v0 is volume of sub-tetrahedron p,v1,v2,v3 / total_volume
    # lambda_0 = Vol(p, v1, v2, v3) / Vol(v0, v1, v2, v3)
    lambda_0 = signed_volume(p, v1, v2, v3) / total_volume
    lambda_1 = signed_volume(v0, p, v2, v3) / total_volume
    lambda_2 = signed_volume(v0, v1, p, v3) / total_volume
    lambda_3 = signed_volume(v0, v1, v2, p) / total_volume
    
    return [lambda_0, lambda_1, lambda_2, lambda_3]
```

**Edge Cases and Pitfalls:**
*   **Degenerate Tetrahedra:** If the tetrahedron volume is zero (coplanar vertices), the division in `compute_barycentric_coordinates` results in undefined behavior. [G] Implementers should check for near-zero volume and handle it as a degenerate case (e.g., fallback to 0 or error).
*   **Precision:** Barycentric coordinate computation involves determinant calculations which can suffer from floating-point precision errors for very small or very large tetrahedra.

## Subtopics

### [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md)
Details the specific geometric computation of the barycentric weights and their affine property within a single element.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, assuming barycentric coordinate computation is constant time.
*   **Memory Complexity:** $O(1)$ additional memory per sample, as data is accessed directly via element ID.
*   **Strengths:** Simple implementation; no need to access neighbor elements or compute neighbor centroids; efficient data access due to duplication during TET-AMR construction.
*   **Limitations:** Only C0 continuity across element boundaries; less accurate than higher-order methods for complex fields.
*   **When to use:** When vertex-centered data is available and high-order continuity is not required.
*   **When not to use:** When cell-centered data is the primary representation and neighbor averaging is preferred, or when higher-order continuity is needed.

## Background
Originating from the work on direct volume rendering of tree-based tetrahedral adaptive mesh refinement (TET-AMR) data, this method leverages the duplicated vertex data structure to simplify sampling [S1, unofficial]. It is part of the broader family of tetrahedral volume rendering techniques that utilize unstructured grids.

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
