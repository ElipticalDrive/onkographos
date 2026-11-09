# Primitive-bound feature embedding

Also called: Feature embedding on virtual scaffold

## Overview
Primitive-bound feature embedding anchors latent feature vectors to a local virtual scaffold that encapsulates each scene primitive, rather than relying on globally fixed grids. This approach solves the scalability and editing limitations of global feature representations by ensuring the feature data moves with the primitives, naturally supporting motion, deformation, and deletion [S1]. It sets itself apart from sibling methods like [Hash Grid Neural Fields](3855-hash-grid-neural-fields.md) or [Explicit Neural Feature Volume](1838-explicit-neural-feature-volume.md) by decoupling the feature storage from the world coordinate system, allowing for high-frequency detail representation within the primitive's support without global memory overhead [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of primitives (e.g., anisotropic 3D Gaussians) defined in world space, each with a position, covariance, and scale.
*   **Output:** A feature vector $\mathbf{f} \in \mathbb{R}^{N_f}$ for any query point $\mathbf{p}$ within a primitive's support, derived from the primitive's local scaffold.

**Data Structures:**
For each primitive $i$, the method maintains a local scaffold structure:
1.  **Canonical Transformation:** A whitening matrix $\mathbf{W}_i$ and translation $\mathbf{t}_i$ that map the primitive's anisotropic ellipsoid in world space to a unit sphere in canonical space.
2.  **Scaffold Vertices:** Four 3D vertices $\mathbf{v}_j \in \mathbb{R}^3$ (for $j \in \{0, 1, 2, 3\}$) defining a tetrahedron that bounds the unit sphere in canonical space.
3.  **Vertex Features:** Four feature vectors $\mathbf{f}_j \in \mathbb{R}^{N_f}$, one associated with each vertex $\mathbf{v}_j$.

**Algorithm Steps:**
1.  **Define Isosurface Ellipsoid:** For an anisotropic 3D Gaussian primitive, define its isosurface ellipsoid with bounded support. This ellipsoid is defined by the primitive's covariance matrix $\Sigma$ [S1].
2.  **Whiten to Canonical Space:** Apply the whitening transformation to map the ellipsoid to a unit sphere in canonical space. The transformation is defined as $\mathbf{x}_{can} = \mathbf{W}(\mathbf{x}_{world} - \mathbf{t})$, where $\mathbf{W}$ is derived from the eigendecomposition of $\Sigma$ [S1].
3.  **Construct Virtual Tetrahedron:** Define a regular tetrahedron in canonical space that bounds the unit sphere. The vertices $\mathbf{v}_j$ of this tetrahedron are fixed relative to the canonical frame [S1].
4.  **Assign Feature Vectors:** Initialize or store a feature vector $\mathbf{f}_j \in \mathbb{R}^{N_f}$ for each of the four vertices $\mathbf{v}_j$ of the tetrahedron. These features are learned parameters associated with the primitive [S1].
5.  **Ray-Primitive Intersection:** For a given ray, determine the intersection points with the primitive's support. Identify the point $\mathbf{p}^*$ along the ray that corresponds to the maximum Gaussian response (density) [S1].
6.  **Transform Query Point:** Transform the query point $\mathbf{p}^*$ from world space to canonical space: $\mathbf{p}^*_{can} = \mathbf{W}(\mathbf{p}^* - \mathbf{t})$ [S1].
7.  **Barycentric Interpolation:** Compute the feature vector $\mathbf{f}$ at $\mathbf{p}^*_{can}$ by performing barycentric interpolation of the vertex features $\mathbf{f}_j$ using the barycentric coordinates of $\mathbf{p}^*_{can}$ with respect to the tetrahedron vertices $\mathbf{v}_j$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_feature(primitive, query_point_world):
    # primitive contains: W (3x3 whitening matrix), t (3x1 translation), 
    # f_vertices (4 x N_f feature vectors), v_vertices (4 x 3 vertex positions in canonical space)
    
    # 1. Transform query point to canonical space
    # p_can = W * (p_world - t)
    diff = query_point_world - primitive.t
    p_can = primitive.W @ diff
    
    # 2. Compute barycentric coordinates of p_can relative to the tetrahedron vertices
    # v0, v1, v2, v3 are primitive.v_vertices
    v0, v1, v2, v3 = primitive.v_vertices
    
    # Calculate volume of the tetrahedron formed by v0, v1, v2, v3
    # Volume = 1/6 * |det(v1-v0, v2-v0, v3-v0)|
    # Barycentric coordinates (lambda0, lambda1, lambda2, lambda3)
    # lambda_i is proportional to the volume of the sub-tetrahedron opposite to vi
    
    # Helper to compute signed volume of tetrahedron defined by 4 points
    def tetra_volume(a, b, c, d):
        return np.dot(a - d, np.cross(b - d, c - d)) / 6.0
    
    vol_total = tetra_volume(v0, v1, v2, v3)
    
    # Compute volumes for sub-tetrahedra
    vol0 = tetra_volume(p_can, v1, v2, v3)
    vol1 = tetra_volume(v0, p_can, v2, v3)
    vol2 = tetra_volume(v0, v1, p_can, v3)
    vol3 = tetra_volume(v0, v1, v2, p_can)
    
    # Barycentric coordinates
    lambda0 = vol0 / vol_total
    lambda1 = vol1 / vol_total
    lambda2 = vol2 / vol_total
    lambda3 = vol3 / vol_total
    
    # 3. Interpolate features
    # f = sum(lambda_j * f_j)
    f = (lambda0 * primitive.f_vertices[0] +
         lambda1 * primitive.f_vertices[1] +
         lambda2 * primitive.f_vertices[2] +
         lambda3 * primitive.f_vertices[3])
         
    return f
```

**Edge Cases and Pitfalls:**
*   **Precision:** Barycentric coordinate calculation involves determinant computations. For degenerate tetrahedra (zero volume), normalization fails. Ensure the tetrahedron is non-degenerate.
*   **Boundary:** The method is only valid for points $\mathbf{p}^*$ inside the primitive's support. If the ray does not intersect the primitive, no feature is retrieved.
*   **Deformation:** Since features are attached to the scaffold which moves with the primitive, no global remapping is needed during motion or deformation [S1].

## Subtopics

### [Neural Harmonic Textures](9123-neural-harmonic-textures.md)
Extends this embedding by applying periodic activations to the interpolated features before compositing, enabling harmonic decomposition and real-time performance. Choose this when high-frequency detail and real-time rendering are required.

## Operating characteristics
*   **Memory:** Memory usage scales linearly with the number of primitives, as each primitive stores only four feature vectors of size $N_f$. This is significantly more efficient than global grids for large scenes or high-frequency details [S1].
*   **Time Complexity:** Feature retrieval is $O(1)$ per primitive per ray intersection, involving a constant number of matrix-vector multiplications and barycentric interpolations.
*   **Strengths:** Supports dynamic scenes (motion, deformation, deletion) without global data structure updates. Scalable to large scenes.
*   **Limitations:** Requires defining a local canonical frame for each primitive. Performance depends on the number of primitives intersected by rays.
*   **When to use:** Ideal for dynamic, deformable, or large-scale scenes where global feature grids are impractical or too memory-intensive.

## Background
Primitive-bound feature embedding was introduced in the context of Neural Harmonic Textures (NHT) for high-quality primitive-based neural reconstruction [S1]. It addresses the limitations of globally defined regular grids, which struggle with scalability and dynamic editing [S1]. The method leverages the Lagrangian nature of primitive-based representations to anchor features locally [S1].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
