# Barycentric Interpolation on Tetrahedra

Also called: 3D Barycentric Coordinates Interpolation, Barycentric Feature Interpolation, Linear Feature Interpolation, Piecewise Linear Interpolation (Tetrahedral)

## Overview
Barycentric interpolation on tetrahedra is a method for reconstructing scalar or vector values at an arbitrary point inside a tetrahedron by computing a weighted sum of the values at the tetrahedron's four vertices, where the weights are the point's 3D barycentric coordinates [S1, unofficial][S2][S3]. It solves the problem of smoothly varying field data across unstructured tetrahedral meshes, which are commonly used to partition irregular volumetric datasets [S2]. Unlike bilinear interpolation on hexahedra, which requires solving a system of equations or iterative refinement, barycentric interpolation on tetrahedra is a direct, closed-form linear operation [S1, unofficial]. It differs from its sibling method, [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md), by operating in 3D volume using four vertices and three independent coordinates, whereas the triangle method operates on 2D surfaces using three vertices and two coordinates [S3].

## How it works
**Inputs:**
- A tetrahedron defined by four vertices $V_1, V_2, V_3, V_4$ with 3D coordinates $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3, \mathbf{v}_4 \in \mathbb{R}^3$.
- A feature vector $\mathbf{f}_i$ associated with each vertex $V_i$ (e.g., scalar density, color, or normal).
- A query point $\mathbf{p} \in \mathbb{R}^3$.

**Outputs:**
- A boolean indicating whether $\mathbf{p}$ lies inside the tetrahedron.
- The interpolated feature vector $\mathbf{f}(\mathbf{p})$.

**Data Structures:**
- Vertices: Array of 4 vectors, each 3 floats.
- Features: Array of 4 vectors, size depends on feature dimension $D$ (e.g., 1 for scalar, 3 for normal).
- Barycentric coordinates: 4 floats $(\alpha_1, \alpha_2, \alpha_3, \alpha_4)$.

**Steps:**

1. **Compute Sub-tetrahedron Volumes:**
   Calculate the signed volume of the four sub-tetrahedra formed by the query point $\mathbf{p}$ and the faces of the original tetrahedron opposite to each vertex. The volume of a tetrahedron with vertices $\mathbf{a}, \mathbf{b}, \mathbf{c}, \mathbf{d}$ is given by $\frac{1}{6} |(\mathbf{b}-\mathbf{a}) \times (\mathbf{c}-\mathbf{a}) \cdot (\mathbf{d}-\mathbf{a})|$ [S3].
   
   Let $V$ be the total volume of the tetrahedron $V_1 V_2 V_3 V_4$.
   Let $V_i$ be the volume of the sub-tetrahedron formed by $\mathbf{p}$ and the face opposite to vertex $i$.
   
   The volumes are computed as [S3]:
   $$
   V_1 = \frac{1}{6} |(\mathbf{v}_2 - \mathbf{p}) \times (\mathbf{v}_3 - \mathbf{p}) \cdot (\mathbf{v}_4 - \mathbf{p})|
   $$
   $$
   V_2 = \frac{1}{6} |(\mathbf{v}_3 - \mathbf{p}) \times (\mathbf{v}_4 - \mathbf{p}) \cdot (\mathbf{v}_1 - \mathbf{p})|
   $$
   $$
   V_3 = \frac{1}{6} |(\mathbf{v}_4 - \mathbf{p}) \times (\mathbf{v}_1 - \mathbf{p}) \cdot (\mathbf{v}_2 - \mathbf{p})|
   $$
   $$
   V_4 = \frac{1}{6} |(\mathbf{v}_1 - \mathbf{p}) \times (\mathbf{v}_2 - \mathbf{p}) \cdot (\mathbf{v}_3 - \mathbf{p})|
   $$
   
   The total volume $V$ is the sum of these sub-volumes:
   $$
   V = V_1 + V_2 + V_3 + V_4
   $$
   
   *Note:* Using absolute values ensures positive volumes for the barycentric weight calculation [S3]. Alternatively, signed volumes can be used if the winding order is consistent, but the condition for being inside requires checking signs [S1, unofficial].

2. **Compute Barycentric Coordinates:**
   The barycentric coordinate $\alpha_i$ for vertex $i$ is the ratio of the sub-volume $V_i$ to the total volume $V$ [S3]:
   $$
   \alpha_i = \frac{V_i}{V}
   $$
   Thus, $\alpha_1 = V_1/V$, $\alpha_2 = V_2/V$, $\alpha_3 = V_3/V$, $\alpha_4 = V_4/V$.
   
   By definition, $\sum_{i=1}^4 \alpha_i = 1$ [S3].

3. **Check Containment:**
   The point $\mathbf{p}$ is inside the tetrahedron if and only if all barycentric coordinates are non-negative [S1, unofficial][S3].
   Condition: $\alpha_1 \ge 0, \alpha_2 \ge 0, \alpha_3 \ge 0, \alpha_4 \ge 0$.
   
   *Equivalence to [S1, unofficial]:* Source [S1, unofficial] uses parameters $\beta, \gamma, \delta$ corresponding to $\alpha_2, \alpha_3, \alpha_4$ respectively, with $\alpha_1 = 1 - \beta - \gamma - \delta$. The condition $\beta > 0, \gamma > 0, \delta > 0$ and $\beta + \gamma + \delta < 1$ is equivalent to $\alpha_2 > 0, \alpha_3 > 0, \alpha_4 > 0$ and $\alpha_1 > 0$ [S1, unofficial].

4. **Interpolate Feature:**
   If the point is inside, compute the interpolated feature $\mathbf{f}(\mathbf{p})$ as the weighted sum of the vertex features [S2][S3]:
   $$
   \mathbf{f}(\mathbf{p}) = \sum_{i=1}^4 \alpha_i \mathbf{f}_i = \alpha_1 \mathbf{f}_1 + \alpha_2 \mathbf{f}_2 + \alpha_3 \mathbf{f}_3 + \alpha_4 \mathbf{f}_4
   $$
   
   *Note:* Source [S1, unofficial] expresses this as:
   $$
   f(p) = (1 - \beta - \gamma - \delta) f(V_{qbcd}) + \beta f(V_{qacd}) + \gamma f(V_{qabd}) + \delta f(V_{qabc})
   $$
   where $V_{qbcd}$ etc. refer to the vertex opposite the face not containing the point in the sub-tetrahedron definition, which maps directly to the $\alpha_i$ formulation [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def barycentric_interpolation_tetrahedron(vertices, features, point):
    """
    Interpolate features at a point inside a tetrahedron using barycentric coordinates.
    
    Args:
        vertices: np.array of shape (4, 3), coordinates of V1, V2, V3, V4.
        features: np.array of shape (4, D), feature vectors at V1, V2, V3, V4.
        point: np.array of shape (3,), query point p.
        
    Returns:
        inside: bool, True if point is inside tetrahedron.
        f_p: np.array of shape (D,), interpolated feature vector.
    """
    v1, v2, v3, v4 = vertices[0], vertices[1], vertices[2], vertices[3]
    
    # Compute vectors from point to vertices
    p_to_v1 = v1 - point
    p_to_v2 = v2 - point
    p_to_v3 = v3 - point
    p_to_v4 = v4 - point
    
    # Compute volumes of sub-tetrahedra
    # V1: opposite to V1, formed by p, V2, V3, V4
    # Volume = 1/6 * |(v2-p) x (v3-p) . (v4-p)|
    vol1 = np.abs(np.dot(np.cross(p_to_v2, p_to_v3), p_to_v4))
    
    # V2: opposite to V2, formed by p, V3, V4, V1
    vol2 = np.abs(np.dot(np.cross(p_to_v3, p_to_v4), p_to_v1))
    
    # V3: opposite to V3, formed by p, V4, V1, V2
    vol3 = np.abs(np.dot(np.cross(p_to_v4, p_to_v1), p_to_v2))
    
    # V4: opposite to V4, formed by p, V1, V2, V3
    vol4 = np.abs(np.dot(np.cross(p_to_v1, p_to_v2), p_to_v3))
    
    total_vol = vol1 + vol2 + vol3 + vol4
    
    # Handle degenerate tetrahedron
    if total_vol < 1e-12:
        return False, np.zeros(features.shape[1])
    
    # Compute barycentric coordinates
    alpha1 = vol1 / total_vol
    alpha2 = vol2 / total_vol
    alpha3 = vol3 / total_vol
    alpha4 = vol4 / total_vol
    
    # Check if inside
    # Point is inside if all alphas are non-negative
    # Due to floating point, we might use a small epsilon
    inside = (alpha1 >= 0) and (alpha2 >= 0) and (alpha3 >= 0) and (alpha4 >= 0)
    
    if not inside:
        return False, np.zeros(features.shape[1])
    
    # Interpolate features
    f_p = alpha1 * features[0] + alpha2 * features[1] + alpha3 * features[2] + alpha4 * features[3]
    
    return True, f_p
```

**Parameters:**
- No tunable parameters. The method is deterministic based on geometry.

**Edge Cases and Pitfalls:**
- **Degenerate Tetrahedra:** If the four vertices are coplanar or coincident, the total volume $V$ is zero. Division by zero must be handled [S3].
- **Precision:** For points very close to the boundary, floating-point errors may cause one $\alpha_i$ to be slightly negative when it should be zero. A small epsilon tolerance may be needed for the containment check [G].
- **Winding Order:** The volume calculation uses absolute values, so winding order does not affect the barycentric weights, provided the point is inside [S3]. If signed volumes were used, consistent winding would be required.

## Subtopics

### [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md)
Barycentric interpolation on triangles estimates vertex attributes at a point within a triangle using 2D barycentric coordinates derived from sub-triangle areas. It is distinct from the tetrahedral method by operating on 2D surfaces with 3 vertices and 2 independent coordinates, resulting in C0 continuity. It is chosen for surface mesh rendering rather than volumetric data sampling. See [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md).

### SDF Aggregation via Barycentric Interpolation
This variant aggregates signed distance function (SDF) values and optional features from new point positions into an existing SDF field using barycentric interpolation within tetrahedra [S4, unofficial]. It is used in neural reconstruction pipelines (e.g., TetWeave) to update volumetric fields with new samples [S4, unofficial]. The method clones SDF values for non-neighbors, computes barycentric coordinates for points inside tetrahedra, and updates the field using a dot product of barycentric weights and tetrahedral SDF values [S4, unofficial]. It is chosen when integrating new geometric samples into a continuous implicit representation [S4, unofficial]. See [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per tetrahedron. The computation involves a constant number of cross products, dot products, and arithmetic operations [S1, unofficial][S3].
- **Memory Complexity:** $O(1)$ auxiliary memory per query, plus storage for the tetrahedral mesh and vertex features [S2].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with the number of tetrahedra traversed. Efficient for sparse traversal but requires efficient mesh traversal structures [S2].
- **Strengths:** Direct, closed-form solution; no iterative refinement; works on unstructured meshes; preserves linearity [S1, unofficial][S2].
- **Limitations:** Only C0 continuous (discontinuous derivatives across faces); less accurate than higher-order methods for smooth fields [S2].
- **When to Use:** Unstructured volumetric data; real-time rendering where simplicity and speed are prioritized over high-order accuracy [S2].
- **When Not to Use:** When C1 continuity or higher-order accuracy is required; for structured grids where trilinear or higher-order interpolation is more efficient [S2].

## Background
Barycentric interpolation on tetrahedra extends the well-known 2D barycentric coordinates to 3D by relating four tetrahedral subvolumes [S1, unofficial]. It was discussed in the context of interactive volume ray tracing as an alternative to cubic polynomial interpolation, which is tedious to derive for tetrahedra [S1, unofficial]. The method is applied within each tetrahedron of an unstructured data set partition to reconstruct in-between values [S2]. It is used in semi-transparent rendering, maximum intensity projection, and iso-surface rendering from ray casting implementations on unstructured grids [S1, unofficial][S2]. The technique is also used in neural reconstruction for aggregating SDF values [S4, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S3] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S4] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
