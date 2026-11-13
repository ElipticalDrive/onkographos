# Primitive-bound feature embedding

Also called: Feature embedding on virtual scaffold

## Overview
Primitive-bound feature embedding anchors latent feature vectors to a local virtual scaffold that encapsulates each geometric primitive, rather than relying on a global static grid [S1]. This approach solves the scalability and editing limitations of globally anchored representations by allowing features to move with primitives during motion, deformation, or deletion [S1]. It sets itself apart from sibling methods like [Explicit Neural Feature Volume](1838-explicit-neural-feature-volume.md) by decoupling feature storage from world-space coordinates, enabling efficient handling of dynamic scenes without global remapping.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of primitives (e.g., anisotropic 3D Gaussians) defined in world space, each with a transformation matrix and covariance.
*   **Output:** A feature vector $\mathbf{f} \in \mathbb{R}^{N_f}$ for any query point $\mathbf{p}^*$ located on a primitive.

**Data Structures:**
For each primitive $i$, the method defines a local virtual scaffold:
1.  **Canonical Space:** The primitive is transformed into its canonical (whitened) space where its support is spherical.
2.  **Bounding Tetrahedron:** A regular tetrahedron is defined to bound the unit sphere in canonical space.
3.  **Vertex Features:** Four feature vectors $\mathbf{f}_j \in \mathbb{R}^{N_f}$ are assigned to the four vertices $j \in \{0, 1, 2, 3\}$ of this tetrahedron. These features are stored locally with the primitive.

**Steps:**
1.  **Whitening:** Transform the query point $\mathbf{x}$ from world space to the primitive's canonical space $\mathbf{x}_{can}$ using the primitive's whitening transformation (inverse of the square root of the covariance matrix) [S1].
2.  **Ray Intersection & Max Response:** For a ray intersecting the primitive, determine the point $\mathbf{p}^*$ along the ray that corresponds to the maximum Gaussian response (density) [S1]. This point lies within the bounded support of the primitive.
3.  **Barycentric Interpolation:** Compute the barycentric coordinates $(\lambda_0, \lambda_1, \lambda_2, \lambda_3)$ of $\mathbf{p}^*$ relative to the vertices of the bounding tetrahedron in canonical space.
4.  **Feature Retrieval:** Interpolate the vertex features to obtain the final feature vector $\mathbf{f}$ at $\mathbf{p}^*$:
    $$ \mathbf{f} = \sum_{j=0}^{3} \lambda_j \mathbf{f}_j $$
    where $\sum \lambda_j = 1$ and $\lambda_j \ge 0$ for points inside the tetrahedron [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_feature(primitive, ray_origin, ray_direction):
    # 1. Find point p* with max Gaussian response along ray
    # This requires solving for the peak of the Gaussian density function
    # along the ray segment within the primitive's support.
    p_star = find_max_gaussian_response(primitive, ray_origin, ray_direction)
    
    # 2. Transform p* to canonical space
    # primitive.whitening_matrix is the inverse sqrt of covariance
    p_canonical = primitive.whitening_matrix @ (p_star - primitive.center)
    
    # 3. Compute barycentric coordinates in the bounding tetrahedron
    # The tetrahedron vertices v_0..v_3 are fixed in canonical space
    # surrounding the unit sphere.
    lambdas = compute_barycentric_coords(p_canonical, primitive.tetrahedron_vertices)
    
    # 4. Interpolate features
    feature = 0.0
    for j in range(4):
        feature += lambdas[j] * primitive.vertex_features[j]
        
    return feature
```

**Edge Cases:**
*   **Point Outside Tetrahedron:** If $\mathbf{p}^*$ falls outside the bounding tetrahedron (though it is inside the sphere), barycentric coordinates may be negative. The method assumes $\mathbf{p}^*$ is within the support where interpolation is valid [S1].
*   **Degenerate Primitives:** If a primitive collapses (zero volume), the whitening transformation is undefined. [G] A minimum variance threshold should be applied to the covariance matrix before whitening.

## Subtopics

### [Neural Harmonic Textures](9123-neural-harmonic-textures.md)
Extends this embedding by applying periodic activations to the interpolated features before decoding, enabling harmonic decomposition and real-time performance. Choose when high-frequency detail and real-time rendering are required.

## Operating characteristics
*   **Memory:** Memory usage scales linearly with the number of primitives, not with scene resolution. Each primitive stores $4 \times N_f$ features.
*   **Performance:** Feature lookup is $O(1)$ per primitive per ray sample, involving a matrix-vector multiplication for whitening and a small linear interpolation.
*   **Strengths:** Supports dynamic editing (motion, deformation, deletion) without global data structure updates. Scalable to large scenes.
*   **Limitations:** Requires defining a local scaffold for each primitive. Interpolation accuracy depends on the tetrahedron's fit to the sphere.

## Background
Originated in the context of primitive-based neural reconstruction to address the limitations of globally defined regular grids [S1]. The method exploits the Lagrangian nature of primitives to anchor features locally, moving with the primitives to support dynamic scene editing [S1]. It bridges the gap between primitive-based and neural-field-based reconstruction by providing a continuous feature field defined per primitive.

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
