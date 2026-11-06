# Vertex-centered Data Sampling


## Overview
Vertex-centered Data Sampling reconstructs scalar field values at arbitrary points within a tetrahedron by interpolating the scalar values stored at the tetrahedron's four vertices using the sample point's barycentric coordinates [S1, unofficial]. It is used when the underlying dataset provides scalar data associated with mesh vertices rather than cell centers [S1, unofficial].

## How it works
**Inputs:**
- The current tetrahedral element $T$ with vertices $V_0, V_1, V_2, V_3$ [G].
- The sample point $P$ inside $T$ [G].
- Vertex-centered scalar values $S_i$ associated with $V_i$ for $i \in \{0, 1, 2, 3\}$ [S1, unofficial].

**Outputs:**
- The interpolated scalar value $S_P$ at point $P$ [G].

**Data Structures:**
- Vertex-centered data is stored duplicated per element during the TET-AMR construction process [S1, unofficial]. This allows direct access to the four vertex scalars via the element ID, without requiring global vertex lookups or neighbor traversal [S1, unofficial].

**Steps:**
1. Compute the barycentric coordinates $(\lambda_0, \lambda_1, \lambda_2, \lambda_3)$ of the sample point $P$ relative to the tetrahedron $T$ [G].
2. Retrieve the four vertex scalar values directly from the element's local data structure using the element ID [S1, unofficial].
3. Calculate the interpolated scalar value $S_P$ using the linear combination of vertex scalars weighted by their respective barycentric coordinates [S1, unofficial].

**Formula:**
$$ S_P = \sum_{i=0}^{3} \lambda_i S_i $$
where:
- $\lambda_i$ is the barycentric coordinate corresponding to vertex $V_i$ [G].
- $S_i$ is the scalar value at vertex $V_i$ [G].
- $\sum_{i=0}^{3} \lambda_i = 1$ and $\lambda_i \ge 0$ for $P$ inside $T$ [G].

**Parameters:**
- Not given in the sources.

**Edge cases and pitfalls:**
- If the sample point lies exactly on a face, edge, or vertex, some barycentric coordinates will be zero [G]. The formula remains valid, but precision issues may arise if coordinates are computed via floating-point arithmetic near boundaries [G].
- The method assumes the scalar data is strictly vertex-centered [S1, unofficial]. For cell-centered data, a different sampling strategy is required [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def vertex_centered_sampling(element_id, sample_point, tetrahedron_vertices, element_scalars):
    """
    Interpolates scalar value at sample_point inside a tetrahedron
    using vertex-centered data stored per element.
    """
    V0, V1, V2, V3 = tetrahedron_vertices
    v0 = V1 - V0
    v1 = V2 - V0
    v2 = V3 - V0
    v3 = sample_point - V0

    d00 = dot(v0, v0); d01 = dot(v0, v1); d02 = dot(v0, v2)
    d11 = dot(v1, v1); d12 = dot(v1, v2); d22 = dot(v2, v2)
    d30 = dot(v3, v0); d31 = dot(v3, v1); d32 = dot(v3, v2)

    det = d00 * (d11 * d22 - d12 * d12) + \
          d01 * (d02 * d12 - d01 * d22) + \
          d02 * (d01 * d12 - d02 * d11)

    inv_det = 1.0 / det
    lambda1 = ((d11 * d22 - d12 * d12) * d30 +
               (d02 * d12 - d01 * d22) * d31 +
               (d01 * d12 - d02 * d11) * d32) * inv_det
    lambda2 = ((d22 * d00 - d02 * d02) * d31 +
               (d02 * d01 - d00 * d12) * d32 +
               (d00 * d12 - d01 * d02) * d30) * inv_det
    lambda3 = ((d00 * d11 - d01 * d01) * d32 +
               (d01 * d12 - d00 * d11) * d30 +
               (d00 * d12 - d01 * d02) * d31) * inv_det
    lambda0 = 1.0 - lambda1 - lambda2 - lambda3

    # Direct access to duplicated vertex data via element ID [S1, unofficial]
    S = element_scalars[element_id]
    S0, S1, S2, S3 = S[0], S[1], S[2], S[3]

    return lambda0 * S0 + lambda1 * S1 + lambda2 * S2 + lambda3 * S3
```

## Subtopics

### [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md)
Barycentric interpolation on tetrahedra computes the scalar value at an arbitrary point inside a tetrahedron by weighting the values at its four vertices according to the point's barycentric coordinates. It solves the problem of accurately sampling vertex-centered data within a tetrahedral mesh, extending the well-known 2D barycentric coordinate system to 3D by relating four tetrahedral subvolumes. Unlike linear interpolation on hexahedral cells which requires solving a system or using trilinear blending, tetrahedral barycentric interpolation provides a direct, geometrically intuitive linear blend based on volume ratios [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a fixed number of arithmetic operations and direct memory access [G].
- **Memory Complexity:** $O(1)$ auxiliary memory per sample, assuming vertex data is pre-duplicated in the element structure [S1, unofficial].
- **Strengths:** Direct access to vertex data without neighbor lookups; linear continuity across tetrahedra; simple implementation [G].
- **Limitations:** Requires vertex-centered data; accuracy depends on the quality of the tetrahedral mesh and the linearity of the underlying field; may exhibit artifacts if the field is highly non-linear within the element [G].
- **When to use:** When the dataset provides vertex-associated scalar values and linear interpolation is sufficient [S1, unofficial].
- **When not to use:** When the data is cell-centered (use [Approximate Cell Sampling](974-approximate-cell-sampling.md) or [Cell-centered Data Sampling](974-approximate-cell-sampling.md#cell-centered-data-sampling)) or when higher-order accuracy is required [G].

## Background
Originating from the work on direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, this method is described in the context of the TET-AMR framework [S1, unofficial]. It leverages the duplication of vertex data during mesh construction to avoid complex global lookups, optimizing performance for rendering [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
