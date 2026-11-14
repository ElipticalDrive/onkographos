# Barycentric Interpolation on Tetrahedra

Also called: 3D Barycentric Coordinates Interpolation, Barycentric Feature Interpolation, Barycentric Interpolation in Tetrahedra, Linear Feature Interpolation, Piecewise Linear Interpolation (Tetrahedral)

## Overview
Barycentric interpolation on tetrahedra reconstructs a feature vector (scalar or multidimensional) at an arbitrary point $p$ inside a tetrahedron by computing a weighted sum of the feature vectors at the tetrahedron's four vertices, where the weights are the barycentric coordinates of $p$ [S1, unofficial][S2][S3][S4]. It extends 2D barycentric interpolation to 3D by relating four tetrahedral subvolumes to the total volume [S1, unofficial]. Unlike its parent method, [Vertex-centered Data Sampling](1037-vertex-centered-data-sampling.md), which describes the general concept of interpolating vertex-centered data, this article details the specific geometric computation of the weights and the resulting affine property within a single tetrahedral element [S3]. It is applied within each tetrahedron of an unstructured data set partition to reconstruct in-between values [S2].

## How it works
**Inputs:**
- Point $p \in \mathbb{R}^3$ (or $q$ in some notations) [S1, unofficial][S3].
- Four vertices of the tetrahedron: $v_0, v_1, v_2, v_3 \in \mathbb{R}^3$ [S3].
- Feature vectors at vertices: $g(v_0), g(v_1), g(v_2), g(v_3) \in \mathbb{R}^d$ (where $d=1$ for scalars) [S3].

**Outputs:**
- Interpolated feature vector $g(p) \in \mathbb{R}^d$ [S3].
- Boolean indicating if $p$ is inside the tetrahedron [S1, unofficial].

**Data Structures:**
- Vertices: Array of 4 vectors, each 3 floats.
- Features: Array of 4 vectors, each $d$ floats.
- Matrix $C$: $3 \times 3$ matrix defined below [S3].
- Matrix $G$: $d \times 3$ matrix defined below [S3].

**Steps:**

1. **Check containment:**
   Compute barycentric coordinates $w_1, w_2, w_3$ (see Step 2). The point $p$ is inside the tetrahedron if and only if:
   $$ w_1 > 0, \quad w_2 > 0, \quad w_3 > 0, \quad \text{and} \quad w_1 + w_2 + w_3 < 1 $$
   [S1, unofficial][S3]. If any condition fails, $p$ is outside. Note: $w_0 = 1 - (w_1 + w_2 + w_3)$, so $w_0 > 0$ is equivalent to $w_1 + w_2 + w_3 < 1$ [S3].

2. **Compute barycentric coordinates:**
   Define the matrix $C \in \mathbb{R}^{3 \times 3}$ using the edge vectors from $v_0$:
   $$ C = \begin{bmatrix} v_1 - v_0 & v_2 - v_0 & v_3 - v_0 \end{bmatrix} $$
   [S3]. Compute the inverse $C^{-1}$.
   Compute the vector $\Delta v = p - v_0$.
   Solve for $[w_1, w_2, w_3]^T$:
   $$ \begin{bmatrix} w_1 \\ w_2 \\ w_3 \end{bmatrix} = C^{-1} (p - v_0) $$
   [S3].
   Compute $w_0$:
   $$ w_0 = 1 - (w_1 + w_2 + w_3) $$
   [S3].

   *Alternative Volume-Based Computation:*
   Barycentric weights can also be computed as ratios of sub-volumes [S4]:
   $$ \alpha_i = \frac{V_i}{V} $$
   where $V$ is the total volume of the tetrahedron, and $V_i$ is the volume of the subtetrahedron formed by $p$ and the face opposite to vertex $i$ [S4].
   For vertices $v_1, v_2, v_3, v_4$ (mapping to $v_0, v_1, v_2, v_3$ above) and point $p$:
   $$ V_1 = \frac{1}{6} | (v_2 - p) \times (v_3 - p) \cdot (v_4 - p) | $$
   $$ V_2 = \frac{1}{6} | (v_3 - p) \times (v_4 - p) \cdot (v_1 - p) | $$
   $$ V_3 = \frac{1}{6} | (v_4 - p) \times (v_1 - p) \cdot (v_2 - p) | $$
   $$ V_4 = \frac{1}{6} | (v_1 - p) \times (v_2 - p) \cdot (v_3 - p) | $$
   [S4]. Note: The absolute value ensures positive volumes, but signed volumes must be used to preserve the sign of the coordinates for containment testing [S1, unofficial][S3].

3. **Interpolate features:**
   Define the matrix $G \in \mathbb{R}^{d \times 3}$ using the feature differences from $v_0$:
   $$ G = \begin{bmatrix} g(v_1) - g(v_0) & g(v_2) - g(v_0) & g(v_3) - g(v_0) \end{bmatrix} $$
   [S3].
   Compute the interpolated feature:
   $$ g(p) = g(v_0) + G \begin{bmatrix} w_1 \\ w_2 \\ w_3 \end{bmatrix} $$
   [S3].
   
   *Expanded Form:*
   $$ g(p) = w_0 g(v_0) + w_1 g(v_1) + w_2 g(v_2) + w_3 g(v_3) $$
   [S3].

   *Affine Form:*
   The interpolation is an affine transformation of the input $p$:
   $$ g(p) = A p + b $$
   [S3], where:
   $$ A = G C^{-1} $$
   $$ b = g(v_0) - G C^{-1} v_0 $$
   [S3]. This form is useful for optimization and analysis but requires precomputing $A$ and $b$ per tetrahedron [S3].

**Parameters:**
- Not given in the sources. The geometry of the tetrahedron and the vertex features are the inputs.

**Edge Cases and Pitfalls:**
- **Degenerate Tetrahedra:** The matrix $C$ is only invertible if the four vertices are not coplanar and no three vertices are collinear [S3]. If degenerate, barycentric coordinate computation fails [S3].
- **Precision:** Floating-point errors may cause $w_i$ to be slightly negative for points on the boundary or inside. A small epsilon tolerance may be needed for containment checks in robust implementations [G].
- **Volume Calculation:** Using absolute values in the volume method loses the sign information required for correct barycentric coordinate signs and containment testing [S1, unofficial][S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def barycentric_interpolation_tetrahedron(p, vertices, features):
    """
    Interpolate features at point p inside a tetrahedron defined by vertices and features.
    
    Args:
        p: np.ndarray, shape (3,), point to evaluate.
        vertices: np.ndarray, shape (4, 3), vertices [v0, v1, v2, v3].
        features: np.ndarray, shape (4, d), feature vectors at vertices.
        
    Returns:
        g_p: np.ndarray, shape (d,), interpolated feature vector.
        is_inside: bool, True if p is inside the tetrahedron.
    """
    v0, v1, v2, v3 = vertices
    g0, g1, g2, g3 = features
    
    # Step 1: Compute matrix C = [v1-v0, v2-v0, v3-v0]
    C = np.column_stack((v1 - v0, v2 - v0, v3 - v0))
    
    # Check for degeneracy (C not invertible)
    if np.abs(np.linalg.det(C)) < 1e-10:
        return np.zeros(features.shape[1]), False
    
    # Step 2: Compute barycentric coordinates w1, w2, w3
    # [w1, w2, w3]^T = C^-1 * (p - v0)
    diff = p - v0
    w123 = np.linalg.solve(C, diff)
    w1, w2, w3 = w123
    
    # w0 = 1 - (w1 + w2 + w3)
    w0 = 1.0 - (w1 + w2 + w3)
    
    # Check containment: w0 > 0, w1 > 0, w2 > 0, w3 > 0
    is_inside = (w0 > 0) and (w1 > 0) and (w2 > 0) and (w3 > 0)
    
    # Step 3: Interpolate features
    # g(p) = w0*g0 + w1*g1 + w2*g2 + w3*g3
    g_p = w0 * g0 + w1 * g1 + w2 * g2 + w3 * g3
    
    return g_p, is_inside
```

## Subtopics

### [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md)
Barycentric interpolation on triangles is the standard method for evaluating vertex attributes (such as colors, normals, or UV coordinates) at an arbitrary point within a triangle, given the values at the triangle's three vertices. It solves the problem of spatially varying attribute representation in mesh-based rendering by providing a linearly varying field over the 2D simplex defined by the triangle. Unlike [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md), which operates in 3D volume using four vertices, this method operates on the 2D surface using three vertices. It is distinct from higher-order methods like [Clough–Tocher Bézier Patch Interpolation](9134-clough-tocher-b-zier-patch-interpolation.md) by being strictly linear and C0 continuous, which is computationally cheaper but prone to visible shading discontinuities across edges if vertex normals are not shared [G].

### SDF Aggregation via Barycentric Interpolation
Aggregates new point positions and their SDF values into the existing SDF field using barycentric interpolation within tetrahedra [S5, unofficial]. The method clones SDF values for non-neighbor points, computes inside tetrahedra and barycentric coordinates for updated points, creates a mask for points inside tetrahedra, and updates SDF at mask positions using einsum: `barycentric_coords[mask] dot sdf[tets[inside_tets[mask]]]` [S5, unofficial]. Feature interpolation is optional and follows the same pattern [S5, unofficial]. Non-neighbor points have SDF and features set to 0 [S5, unofficial]. Positions are updated for non-neighbor points [S5, unofficial].

### Input Preconditioning for Tetrahedral Encoder
A global affine transformation $y = A^* x + t$ applied to input coordinates before encoding, designed to mitigate geometric anisotropy introduced by tetrahedral barycentric interpolation [S3]. The preconditioner $T$ in homogeneous coordinates is $T = [[A^*, t], [0^T, 1]]$ [S3]. The encoder-induced local metric for tetrahedron $T$ is $M_T = C_T^{-1} C_T^{-1}$, where $C_T = [v_1-v_0, v_2-v_0, v_3-v_0]$ [S3]. The cube-averaged metric $M = \frac{1}{6} \sum_{T \in \{S0..S5\}} M_T = \frac{1}{3} [[5, -2, -2], [-2, 5, -2], [-2, -2, 5]]$ [S3]. $A^* \in \mathbb{R}^{3 \times 3}$ must be invertible [S3]. Without preconditioning, the encoder attenuates motion along the body diagonal $[1,1,1]$ and amplifies transverse directions by $\sqrt{7}$ on average, causing directional bias in feature interpolation [S3].

### Levelwise Feature Concatenation
Concatenates barycentrically interpolated feature vectors from multiple resolution levels $\ell = 0$ to $L-1$ into a single feature vector $z(x)$ [S3]. The formula is $z(x) = \bigoplus_{\ell=0}^{L-1} g^{(\ell)}(x) = [g^{(0)}(x)^T, \dots, g^{(L-1)}(x)^T]^T \in \mathbb{R}^{dL}$ [S3]. In a region where feature selection is fixed, $z(x) = A' x + b'$, where $A'$ stacks $A^{(\ell)}$ and $b'$ stacks $b^{(\ell)}$ [S3]. This preserves the affine property of the interpolation across levels [S3].

### Barycentric Feature Interpolation for Neural Textures
A sampling technique for neural proxy volume rendering that maps Cartesian coordinates to Barycentric coordinates to sample features from a neural texture, using local information instead of global scene coordinates [S6, unofficial]. It decomposes proxy prisms into three tetrahedra using the method from [PBFJ05] [S6, unofficial]. For a Cartesian point $x$ and tetrahedron $\tau$ with vertices $ABCD$, it computes volumes $V_A, V_B, V_C, V_D$ and $V_{ABCD}$, then checks if sample $x$ is within $\tau$ by verifying $\phi_A, \phi_B, \phi_C, \phi_D \ge 0$ [S6, unofficial]. It finds the specific tetrahedron $\tau_x$ containing $x$, and computes UV coordinate $u$ from Barycentric coordinates using vertex UVs: $u = \phi_A u_A + \phi_B u_B + \phi_C u_C + \phi_D u_D$ [S6, unofficial]. Parameters include $\theta_{Tex}$ for the learned network $S_{Tex}$ and $\theta_p$ for the proxy sampler [S6, unofficial].

## Operating characteristics
**Time Complexity:**
- Computing barycentric coordinates: $O(1)$ for a fixed-size $3 \times 3$ matrix inversion or solve [S3].
- Interpolating features: $O(d)$ where $d$ is the feature dimension [S3].
- Overall per-point: $O(1)$ assuming fixed tetrahedron size and feature dimension [S3].

**Memory Complexity:**
- Storage per tetrahedron: $O(1)$ for vertices and features [S3].
- No additional global memory required for the interpolation itself [S3].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Linear with the number of points evaluated [S3].
- Linear with the feature dimension $d$ [S3].

**Strengths:**
- Simple to implement [G].
- Computationally efficient [G].
- Provides C0 continuous interpolation [G].
- Affine property enables analytical derivatives and theoretical analysis [S3].

**Limitations:**
- Only C0 continuous; discontinuities at tetrahedron boundaries [G].
- Sensitive to tetrahedron quality; degenerate tetrahedra cause numerical instability [S3].
- Geometric anisotropy in regular grids can cause directional bias in feature interpolation without preconditioning [S3].

**When to use:**
- When C0 continuity is sufficient [G].
- When computational efficiency is critical [G].
- When working with unstructured tetrahedral meshes [S2].

**When not to use:**
- When higher-order continuity (C1 or higher) is required [G].
- When tetrahedra are likely to be degenerate [S3].
- When directional bias in regular grids is unacceptable without preconditioning [S3].

## Background
Barycentric interpolation on tetrahedra extends the well-known 2D barycentric coordinates to 3D by relating four tetrahedral subvolumes [S1, unofficial]. It is used for piecewise linear interpolation within unstructured data sets partitioned into tetrahedral meshes [S2]. The method is fundamental in tetrahedral volume rendering, where it reconstructs values at sample points along rays [S1, unofficial][S2]. It is also used in feature interpolation for primitive-based neural reconstruction [S4], and in SDF aggregation methods like TetWeave [S5, unofficial]. The affine property of barycentric interpolation within a tetrahedron has been formally proven and is used in tetrahedral networks to show piecewise affine behavior [S3]. Input preconditioning was introduced to mitigate geometric anisotropy in tetrahedral encoders [S3].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S3] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
[S4] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S5] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
[S6] Neural Proxy: Empowering Neural Volume Rendering for Animation, https://doi.org/10.2312/pg.20211384, fetched 2026-10-26 (unofficial)
