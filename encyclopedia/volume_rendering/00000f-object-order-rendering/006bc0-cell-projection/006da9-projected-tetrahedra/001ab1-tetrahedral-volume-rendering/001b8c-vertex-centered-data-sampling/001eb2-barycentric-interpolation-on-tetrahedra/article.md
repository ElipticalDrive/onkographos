# Barycentric Interpolation on Tetrahedra

Also called: 3D Barycentric Coordinates Interpolation, Barycentric Feature Interpolation, Barycentric Interpolation in Tetrahedra, Linear Feature Interpolation, Piecewise Linear Interpolation (Tetrahedral)

## Overview
Barycentric interpolation on tetrahedra reconstructs a scalar or vector field value at an arbitrary point $p$ inside a tetrahedron by computing a weighted sum of the values at the tetrahedron's four vertices, where the weights are the point's barycentric coordinates [S1, unofficial][S2, S3, S4]. It extends 2D barycentric interpolation to the 3D simplex, providing a piecewise linear, C0-continuous field over unstructured tetrahedral meshes [S1, unofficial][S2]. Unlike [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md), which operates on 2D surfaces, this method extends 2D barycentric coordinates to the 3D simplex using four vertices [S1, unofficial].

## How it works
**Inputs:**
- Point $p \in \mathbb{R}^3$ (or $\mathbf{x}$) [S3, S4].
- Four vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3 \in \mathbb{R}^3$ defining the tetrahedron [S3, S4].
- Feature vectors $\mathbf{g}(\mathbf{v}_i) \in \mathbb{R}^d$ (or scalar values $f(\mathbf{v}_i)$) associated with each vertex [S3, S4].

**Outputs:**
- Interpolated feature vector $\mathbf{g}(p) \in \mathbb{R}^d$ (or scalar $f(p)$) [S3, S4].
- Boolean indicating if $p$ is inside the tetrahedron [S1, unofficial].

**Data Structures:**
- Vertices: Array of 4 vectors, each 3 floats.
- Features: Array of 4 vectors, each $d$ floats.
- Matrix $C$: $3 \times 3$ matrix where column $j$ is $\mathbf{v}_{j+1} - \mathbf{v}_0$ for $j=0,1,2$ [S3].
- Matrix $G$: $d \times 3$ matrix where column $j$ is $\mathbf{g}(\mathbf{v}_{j+1}) - \mathbf{g}(\mathbf{v}_0)$ for $j=0,1,2$ [S3].

**Steps:**

1. **Check containment:** Determine if $p$ lies inside the tetrahedron. The point is inside if and only if all barycentric coordinates $\beta, \gamma, \delta$ (defined below) are positive and their sum is less than 1 [S1, unofficial]. Alternatively, compute volumes as in [S4][S6, unofficial] and check if all partial volumes are non-negative [S6, unofficial].

2. **Compute barycentric coordinates:**
   There are two equivalent methods to compute the coordinates [S3, S4][S6, unofficial].
   
   *Method A: Volume Ratio (Geometric)*
   Compute the total volume $V$ and partial volumes $V_i$ [S4][S6, unofficial]:
   $$ V = \frac{1}{6} |(\mathbf{v}_1 - \mathbf{v}_0) \cdot ((\mathbf{v}_2 - \mathbf{v}_0) \times (\mathbf{v}_3 - \mathbf{v}_0))| $$
   Using the explicit vertex mapping from [S6, unofficial] for tetrahedron vertices $A, B, C, D$ (mapped to $v_0, v_1, v_2, v_3$):
   $$ V_A = \frac{1}{6} (p - \mathbf{v}_1) \cdot ((\mathbf{v}_3 - \mathbf{v}_1) \times (\mathbf{v}_2 - \mathbf{v}_1)) $$
   $$ V_B = \frac{1}{6} (p - \mathbf{v}_0) \cdot ((\mathbf{v}_2 - \mathbf{v}_0) \times (\mathbf{v}_3 - \mathbf{v}_0)) $$
   $$ V_C = \frac{1}{6} (p - \mathbf{v}_0) \cdot ((\mathbf{v}_3 - \mathbf{v}_0) \times (\mathbf{v}_1 - \mathbf{v}_0)) $$
   $$ V_D = \frac{1}{6} (p - \mathbf{v}_0) \cdot ((\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0)) $$
   $$ V_{total} = \frac{1}{6} (\mathbf{v}_1 - \mathbf{v}_0) \cdot ((\mathbf{v}_2 - \mathbf{v}_0) \times (\mathbf{v}_3 - \mathbf{v}_0)) $$
   The barycentric weights are $\phi_A = V_A / V_{total}$, $\phi_B = V_B / V_{total}$, etc. [S6, unofficial].
   
   *Method B: Linear Algebra (Algebraic)*
   Define matrix $C = [\mathbf{v}_1 - \mathbf{v}_0, \mathbf{v}_2 - \mathbf{v}_0, \mathbf{v}_3 - \mathbf{v}_0] \in \mathbb{R}^{3 \times 3}$ [S3].
   Solve for $w_1, w_2, w_3$:
   $$ \begin{bmatrix} w_1 \\ w_2 \\ w_3 \end{bmatrix} = C^{-1} (p - \mathbf{v}_0) $$
   Then $w_0 = 1 - (w_1 + w_2 + w_3)$ [S3].

3. **Interpolate features:**
   Compute the weighted sum:
   $$ \mathbf{g}(p) = w_0 \mathbf{g}(\mathbf{v}_0) + w_1 \mathbf{g}(\mathbf{v}_1) + w_2 \mathbf{g}(\mathbf{v}_2) + w_3 \mathbf{g}(\mathbf{v}_3) $$
   [S3].
   
   Alternatively, using the affine property [S3]:
   $$ \mathbf{g}(p) = \mathbf{g}(\mathbf{v}_0) + G C^{-1} (p - \mathbf{v}_0) $$
   where $G = [\mathbf{g}(\mathbf{v}_1) - \mathbf{g}(\mathbf{v}_0), \mathbf{g}(\mathbf{v}_2) - \mathbf{g}(\mathbf{v}_0), \mathbf{g}(\mathbf{v}_3) - \mathbf{g}(\mathbf{v}_0)]$ [S3].
   
   Or in expanded form [S3]:
   $$ \mathbf{g}(p) = \mathbf{g}(\mathbf{v}_0) + (\mathbf{g}(\mathbf{v}_1) - \mathbf{g}(\mathbf{v}_0))w_1 + (\mathbf{g}(\mathbf{v}_2) - \mathbf{g}(\mathbf{v}_0))w_2 + (\mathbf{g}(\mathbf{v}_3) - \mathbf{g}(\mathbf{v}_0))w_3 $$

**Parameters:**
Not given in the sources.

**Edge cases and pitfalls:**
- **Degeneracy:** The matrix $C$ is only invertible if the four vertices are not coplanar and no three vertices are collinear [S3]. If degenerate, barycentric coordinate computation fails [S3].
- **Precision:** Computing volumes using cross products can suffer from precision issues for very small or very large tetrahedra [G].
- **Boundary:** Points on the boundary have one or more barycentric coordinates equal to 0. Points outside have at least one negative coordinate [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def barycentric_interpolation_tetrahedron(p, vertices, features):
    """
    Interpolate features at point p inside a tetrahedron defined by vertices.
    
    Args:
        p: numpy array, shape (3,), point to evaluate.
        vertices: numpy array, shape (4, 3), vertices [v0, v1, v2, v3].
        features: numpy array, shape (4, d), feature vectors at vertices.
        
    Returns:
        interpolated_feature: numpy array, shape (d,), interpolated value.
        is_inside: bool, True if p is inside the tetrahedron.
    """
    v0, v1, v2, v3 = vertices
    g0, g1, g2, g3 = features
    
    # Construct matrix C: columns are v1-v0, v2-v0, v3-v0
    C = np.column_stack((v1 - v0, v2 - v0, v3 - v0))
    
    # Check if C is invertible (determinant non-zero)
    det_C = np.linalg.det(C)
    if np.isclose(det_C, 0):
        raise ValueError("Degenerate tetrahedron: vertices are coplanar or collinear.")
    
    # Compute barycentric coordinates w1, w2, w3
    # Solve C * [w1, w2, w3]^T = p - v0
    diff = p - v0
    try:
        w123 = np.linalg.solve(C, diff)
    except np.linalg.LinAlgError:
        raise ValueError("Matrix C is singular.")
        
    w1, w2, w3 = w123
    w0 = 1.0 - (w1 + w2 + w3)
    
    # Check if point is inside
    # Inside if w0, w1, w2, w3 >= 0 (with some epsilon for float precision)
    epsilon = 1e-9
    is_inside = (w0 >= -epsilon) and (w1 >= -epsilon) and (w2 >= -epsilon) and (w3 >= -epsilon)
    
    if not is_inside:
        # Depending on application, might return None or extrapolate.
        # Standard barycentric interpolation is only defined inside.
        return None, False

    # Compute interpolated feature
    # g(p) = w0*g0 + w1*g1 + w2*g2 + w3*g3
    interpolated_feature = w0 * g0 + w1 * g1 + w2 * g2 + w3 * g3
    
    return interpolated_feature, True

def barycentric_interpolation_tetrahedron_affine(p, vertices, features):
    """
    Alternative implementation using the affine property g(x) = Ax + b.
    Precomputes A and b for a specific tetrahedron and feature set.
    """
    v0, v1, v2, v3 = vertices
    g0, g1, g2, g3 = features
    
    # Construct C and G
    C = np.column_stack((v1 - v0, v2 - v0, v3 - v0))
    G = np.column_stack((g1 - g0, g2 - g0, g3 - g0))
    
    # Compute A = G * C^-1
    C_inv = np.linalg.inv(C)
    A = G @ C_inv
    
    # Compute b = g0 - G * C^-1 * v0
    b = g0 - G @ C_inv @ v0
    
    # Check containment using barycentric coords derived from x = v0 + C*w
    diff = p - v0
    w123 = C_inv @ diff
    w1, w2, w3 = w123
    w0 = 1.0 - (w1 + w2 + w3)
    
    epsilon = 1e-9
    is_inside = (w0 >= -epsilon) and (w1 >= -epsilon) and (w2 >= -epsilon) and (w3 >= -epsilon)
    
    if not is_inside:
        return None, False
        
    # g(p) = A @ p + b
    interpolated_feature = A @ p + b
    
    return interpolated_feature, True
```

## Subtopics

### [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md)
Barycentric interpolation on triangles evaluates vertex attributes (normals, colors, UVs) at a ray hit point using barycentric coordinates. It provides a linearly varying field over the 2D simplex defined by the triangle. Unlike [Barycentric Interpolation on Tetrahedra](1843-barycentric-interpolation-on-tetrahedra.md), which operates in 3D volume using four vertices, this method operates on the 2D surface using three vertices. It is distinct from [Clough–Tocher Bézier Patch Interpolation](9134-clough-tocher-b-zier-patch-interpolation.md) by being strictly linear and C0 continuous, which is computationally cheaper but prone to visible shading discontinuities across edges if vertex normals are not shared.

### SDF Aggregation via Barycentric Interpolation
Aggregates new point positions and their SDF values into the existing SDF field using barycentric interpolation within tetrahedra [S5, unofficial]. It clones SDF values for non-neighbor points, computes inside tetrahedra and barycentric coordinates for updated points, creates a mask for points inside tetrahedra, and updates SDF at mask positions using einsum: `barycentric_coords[mask] dot sdf[tets[inside_tets[mask]]]` [S5, unofficial]. Optional feature interpolation follows the same pattern [S5, unofficial]. Non-neighbor points have SDF and features set to 0, and positions are updated [S5, unofficial].

### Input Preconditioning for Tetrahedral Encoder
A global affine transformation $y = A^* x + t$ applied to input coordinates before encoding, designed to mitigate geometric anisotropy introduced by tetrahedral barycentric interpolation [S3]. The preconditioner $T$ in homogeneous coordinates is $T = \begin{bmatrix} A^* & t \\ 0^T & 1 \end{bmatrix}$ [S3]. The encoder-induced local metric for tetrahedron $T$ is $M_T = C_T^{-1} C_T^{-1}$, where $C_T = [v_1-v_0, v_2-v_0, v_3-v_0]$ [S3]. The cube-averaged metric $M = \frac{1}{6} \sum_{T \in \{S_0..S_5\}} M_T = \frac{1}{3} \begin{bmatrix} 5 & -2 & -2 \\ -2 & 5 & -2 \\ -2 & -2 & 5 \end{bmatrix}$ is invariant under coordinate axis permutations [S3]. $A^* \in \mathbb{R}^{3 \times 3}$ must be invertible. Without preconditioning, the encoder attenuates motion along the body diagonal $[1,1,1]$ and amplifies transverse directions by $\sqrt{7}$ on average, causing directional bias in feature interpolation [S3].

### Levelwise Feature Concatenation
Concatenates barycentrically interpolated feature vectors from multiple resolution levels $\ell = 0$ to $L-1$ into a single feature vector $z(x)$ [S3]. The formula is $z(x) = \bigoplus_{\ell=0}^{L-1} g^{(\ell)}(x) = [g^{(0)}(x)^T, \dots, g^{(L-1)}(x)^T]^T \in \mathbb{R}^{dL}$ [S3]. In a region where feature selection is fixed, $z(x) = A'x + b'$, where $A'$ stacks $A^{(\ell)}$ and $b'$ stacks $b^{(\ell)}$ [S3].

### Barycentric Feature Interpolation for Neural Textures
A sampling technique for neural proxy volume rendering that maps Cartesian coordinates to Barycentric coordinates to sample features from a neural texture, using local information instead of global scene coordinates [S6, unofficial]. It decomposes proxy prisms into three tetrahedra using the method from [S6, unofficial]. For a Cartesian point $x$ and tetrahedron $\tau$ with vertices $ABCD$, it computes volumes $V_A, V_B, V_C, V_D$ and $V_{ABCD}$ to derive barycentric coordinates $\phi_A, \phi_B, \phi_C, \phi_D$ [S6, unofficial]. It checks if sample $x$ is within tetrahedron $\tau$ by verifying $\phi_A, \phi_B, \phi_C, \phi_D \ge 0$ [S1, unofficial][S6, unofficial]. It finds the specific tetrahedron $\tau_x$ containing $x$. It computes UV coordinate $u$ from Barycentric coordinates using vertex UVs: $u = \phi_A u_A + \phi_B u_B + \phi_C u_C + \phi_D u_D$ [S6, unofficial]. Parameters include $\theta_{Tex}$ for the learned network $S_{Tex}$ and $\theta_p$ for the proxy sampler $\hat{S}_p$ [S6, unofficial].

## Operating characteristics
**Time and memory complexity:**
- Time: $O(1)$ per tetrahedron for coordinate computation and interpolation, assuming fixed dimension $d$. Matrix inversion of $3 \times 3$ is constant time.
- Memory: $O(1)$ auxiliary memory per tetrahedron for storing $C$ and $G$ matrices if precomputed.

**Measured performance:**
Not given in the sources.

**Scaling:**
- Linear with the number of tetrahedra traversed.

**Strengths and limitations:**
- Strengths: Simple, computationally efficient, piecewise linear, C0 continuous [S1, unofficial][S2].
- Limitations: Prone to visible shading discontinuities across edges if vertex normals are not shared (inherited from linear interpolation) [S1, unofficial]. Sensitive to geometric anisotropy in tetrahedral meshes without preconditioning [S3]. Degeneracy issues if vertices are coplanar [S3].

**When to use it and when not:**
- Use for volumetric data sampling in unstructured tetrahedral meshes [S2].
- Use when computational efficiency is critical and C0 continuity is acceptable [S1, unofficial].
- Do not use if high-order continuity (C1 or higher) is required; consider [Clough–Tocher Bézier Patch Interpolation](9134-clough-tocher-b-zier-patch-interpolation.md) instead [S1, unofficial].
- Do not use on degenerate tetrahedra [S3].

## Background
Barycentric interpolation on tetrahedra extends the well-known 2D barycentric coordinates to 3D by relating four tetrahedral subvolumes [S1, unofficial]. It is applied within each tetrahedron of an unstructured data set partition to reconstruct in-between values as a weighted sum of scalar values at the vertices [S2]. The method is fundamental in [Tetrahedral Volume Rendering](1843-tetrahedral-volume-rendering.md) and [Vertex-centered Data Sampling](1037-vertex-centered-data-sampling.md). It has been used in interactive volume ray tracing [S1, unofficial], unstructured data set rendering [S2], and more recently in neural representations like TetraSDF [S3], Neural Harmonic Textures [S4], TetWeave [S5, unofficial], and Neural Proxy [S6, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S3] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
[S4] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S5] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
[S6] Neural Proxy: Empowering Neural Volume Rendering for Animation, https://doi.org/10.2312/pg.20211384, fetched 2026-10-26 (unofficial)
