# Clough–Tocher Bézier Patch Interpolation

Also called: C1 continuous Bézier patch, Clough-Tocher scheme, Clough–Tocher C1 Interpolation, Cubic Feature Interpolation

## Overview
Clough–Tocher Bézier Patch Interpolation replaces linear barycentric interpolation with a cubic Bézier patch construction to interpolate feature values and gradients over a triangle, guaranteeing C1 continuity everywhere and C∞ within each triangle [S1]. It solves the problem of visible seam artifacts caused by gradient discontinuities across triangle edges in fully opaque meshes, which arise because standard barycentric interpolation produces a C0 feature field [S1]. Unlike its parent method, [Barycentric Interpolation on Triangles](5671-barycentric-interpolation-on-triangles.md), which only uses vertex values, this method requires per-vertex gradients to define the patch geometry [S1].

## How it works
**Inputs:**
*   Triangle vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$.
*   Feature values at vertices $f_0, f_1, f_2$.
*   Feature gradients at vertices $\nabla f_0, \nabla f_1, \nabla f_2$.
*   Evaluation point $\mathbf{p}$ in 3D space (used to compute barycentric coordinates).

**Outputs:**
*   Interpolated feature value $f(\mathbf{p})$.
*   (Optional) Interpolated gradient $\nabla f(\mathbf{p})$ if the derivative of the Bézier patch is required.

**Data Structures:**
*   **Bézier Control Points:** An array of 10 scalar (or vector) values $c_{ijk}$ where $i+j+k=3$. The layout is:
    *   Vertices: $c_{300}, c_{030}, c_{003}$
    *   Edges: $c_{210}, c_{120}, c_{021}, c_{012}, c_{102}, c_{201}$
    *   Center: $c_{111}$

**Steps:**

1.  **Compute Edge Vectors:**
    Calculate the edge vectors for the triangle:
    $$ \mathbf{e}_{01} = \mathbf{v}_1 - \mathbf{v}_0 $$
    $$ \mathbf{e}_{12} = \mathbf{v}_2 - \mathbf{v}_1 $$
    $$ \mathbf{e}_{20} = \mathbf{v}_0 - \mathbf{v}_2 $$

2.  **Compute Bézier Control Points:**
    Define the 10 control points using the vertex values, gradients, and edge vectors [S1]:
    $$ c_{300} = f_0 $$
    $$ c_{030} = f_1 $$
    $$ c_{003} = f_2 $$
    $$ c_{210} = f_0 + \frac{1}{3} \nabla f_0 \cdot \mathbf{e}_{01} $$
    $$ c_{120} = f_1 - \frac{1}{3} \nabla f_1 \cdot \mathbf{e}_{01} $$
    $$ c_{021} = f_1 + \frac{1}{3} \nabla f_1 \cdot \mathbf{e}_{12} $$
    $$ c_{012} = f_2 - \frac{1}{3} \nabla f_2 \cdot \mathbf{e}_{12} $$
    $$ c_{102} = f_2 + \frac{1}{3} \nabla f_2 \cdot \mathbf{e}_{20} $$
    $$ c_{201} = f_0 - \frac{1}{3} \nabla f_0 \cdot \mathbf{e}_{20} $$
    $$ c_{111} = \frac{1}{6} (c_{210} + c_{120} + c_{021} + c_{012} + c_{102} + c_{201}) - \frac{1}{6} (c_{300} + c_{030} + c_{003}) $$

3.  **Compute Barycentric Coordinates:**
    Calculate the barycentric coordinates $(\alpha_0, \alpha_1, \alpha_2)$ of the evaluation point $\mathbf{p}$ relative to the triangle vertices [S1]. This is typically done by projecting $\mathbf{p}$ onto the triangle plane or using the area method. Let $\alpha_i$ be the weight for vertex $i$.

4.  **Evaluate Bézier Patch:**
    Compute the interpolated feature value $f(\mathbf{p})$ using the Bernstein basis polynomials of degree 3 [S1]:
    $$ f(\mathbf{p}) = \sum_{i+j+k=3} \binom{3}{i,j,k} \alpha_0^i \alpha_1^j \alpha_2^k c_{ijk} $$
    where $\binom{3}{i,j,k} = \frac{3!}{i!j!k!}$ is the multinomial coefficient.

**Gradient Estimation (Preprocessing):**
The method requires per-vertex gradients $\nabla f_v$. These are estimated via a per-vertex least-squares fit over the one-ring neighborhood [S1]. For each vertex $v$, gather all neighbor vertices $\{v_j\}_{j \in N(v)}$ and solve the 2x2 linear system:
$$ \nabla f_v = \mathbf{A}_v^{-1} \mathbf{b}_v $$
where:
$$ \mathbf{A}_v = \sum_j \Delta \mathbf{p}_j \Delta \mathbf{p}_j^\top $$
$$ \mathbf{b}_v = \sum_j \Delta f_j \Delta \mathbf{p}_j $$
Here, $\Delta \mathbf{p}_j = \mathbf{v}_j - \mathbf{v}_v$ and $\Delta f_j = f_j - f_v$. The sum is over all neighbors $j$ of vertex $v$. The 2x2 system is solved analytically [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_barycentric_coords(v0, v1, v2, p):
    """
    Computes barycentric coordinates of point p relative to triangle v0, v1, v2.
    Assumes p is coplanar with the triangle or projected onto it.
    Returns (alpha0, alpha1, alpha2).
    """
    # Vector from v0 to vertices
    v0v1 = v1 - v0
    v0v2 = v2 - v0
    v0p = p - v0
    
    # Dot products
    dot00 = dot(v0v2, v0v2)
    dot01 = dot(v0v2, v0v1)
    dot02 = dot(v0v2, v0p)
    dot11 = dot(v0v1, v0v1)
    dot12 = dot(v0v1, v0p)
    
    # Inverse determinant
    inv_det = 1.0 / (dot00 * dot11 - dot01 * dot01)
    
    # Barycentric coordinates
    alpha2 = (dot11 * dot02 - dot01 * dot12) * inv_det
    alpha1 = (dot00 * dot12 - dot01 * dot02) * inv_det
    alpha0 = 1.0 - alpha1 - alpha2
    
    return alpha0, alpha1, alpha2

def clough_tocher_interpolate(v0, v1, v2, f0, f1, f2, grad0, grad1, grad2, p):
    """
    Interpolates feature value at point p using Clough-Tocher Bézier patch.
    
    Args:
        v0, v1, v2: Vertex positions (3D vectors)
        f0, f1, f2: Feature values at vertices (scalars or vectors)
        grad0, grad1, grad2: Feature gradients at vertices (2D or 3D vectors depending on context)
        p: Evaluation point (3D vector)
        
    Returns:
        Interpolated feature value
    """
    # 1. Compute Edge Vectors
    e01 = v1 - v0
    e12 = v2 - v1
    e20 = v0 - v2
    
    # 2. Compute Control Points
    # Vertex values
    c300 = f0
    c030 = f1
    c003 = f2
    
    # Edge-related points
    # Note: dot product of gradient and edge vector
    c210 = f0 + (1.0/3.0) * dot(grad0, e01)
    c120 = f1 - (1.0/3.0) * dot(grad1, e01)
    c021 = f1 + (1.0/3.0) * dot(grad1, e12)
    c012 = f2 - (1.0/3.0) * dot(grad2, e12)
    c102 = f2 + (1.0/3.0) * dot(grad2, e20)
    c201 = f0 - (1.0/3.0) * dot(grad0, e20)
    
    # Center point
    sum_edges = c210 + c120 + c021 + c012 + c102 + c201
    sum_vertices = c300 + c030 + c003
    c111 = (1.0/6.0) * sum_edges - (1.0/6.0) * sum_vertices
    
    # 3. Compute Barycentric Coordinates
    alpha0, alpha1, alpha2 = compute_barycentric_coords(v0, v1, v2, p)
    
    # 4. Evaluate Bézier Patch
    # Bernstein basis polynomials for degree 3
    # B_300 = alpha0^3
    # B_030 = alpha1^3
    # B_003 = alpha2^3
    # B_210 = 3 * alpha0^2 * alpha1
    # B_120 = 3 * alpha0 * alpha1^2
    # B_021 = 3 * alpha1^2 * alpha2
    # B_012 = 3 * alpha1 * alpha2^2
    # B_102 = 3 * alpha0^2 * alpha2
    # B_201 = 3 * alpha0 * alpha2^2
    # B_111 = 6 * alpha0 * alpha1 * alpha2
    
    alpha0_sq = alpha0 * alpha0
    alpha1_sq = alpha1 * alpha1
    alpha2_sq = alpha2 * alpha2
    
    b300 = alpha0_sq * alpha0
    b030 = alpha1_sq * alpha1
    b003 = alpha2_sq * alpha2
    
    b210 = 3.0 * alpha0_sq * alpha1
    b120 = 3.0 * alpha0 * alpha1_sq
    b021 = 3.0 * alpha1_sq * alpha2
    b012 = 3.0 * alpha1 * alpha2_sq
    b102 = 3.0 * alpha0_sq * alpha2
    b201 = 3.0 * alpha0 * alpha2_sq
    
    b111 = 6.0 * alpha0 * alpha1 * alpha2
    
    # Sum contributions
    result = (b300 * c300 + 
              b030 * c030 + 
              b003 * c003 + 
              b210 * c210 + 
              b120 * c120 + 
              b021 * c021 + 
              b012 * c012 + 
              b102 * c102 + 
              b201 * c201 + 
              b111 * c111)
              
    return result

def estimate_vertex_gradient(v_center, neighbors_pos, neighbors_feat, center_feat):
    """
    Estimates gradient at a vertex using least-squares fit over one-ring neighborhood.
    
    Args:
        v_center: Position of the center vertex
        neighbors_pos: List of positions of neighbor vertices
        neighbors_feat: List of feature values of neighbor vertices
        center_feat: Feature value of the center vertex
        
    Returns:
        Gradient vector (2D or 3D)
    """
    # Initialize A (2x2 or 3x3) and b (2x or 3x1)
    # Assuming 2D gradients for planar triangles embedded in 3D, 
    # or 3D gradients if features are volumetric. 
    # The source specifies a 2x2 system, implying gradients are defined in the local tangent plane 
    # or the features are 2D. We assume a 2D gradient context for the 2x2 matrix.
    
    A = [[0.0, 0.0], [0.0, 0.0]]
    b = [0.0, 0.0]
    
    for pos, feat in zip(neighbors_pos, neighbors_feat):
        dp = pos - v_center # Delta p
        df = feat - center_feat # Delta f
        
        # A += dp * dp^T
        A[0][0] += dp[0] * dp[0]
        A[0][1] += dp[0] * dp[1]
        A[1][0] += dp[1] * dp[0]
        A[1][1] += dp[1] * dp[1]
        
        # b += df * dp
        b[0] += df * dp[0]
        b[1] += df * dp[1]
        
    # Solve 2x2 system analytically: A * grad = b
    det = A[0][0] * A[1][1] - A[0][1] * A[1][0]
    
    # Handle singular matrix (degenerate case)
    if abs(det) < 1e-8:
        return [0.0, 0.0] # [G] Fallback to zero gradient if system is singular
        
    inv_det = 1.0 / det
    grad0 = (A[1][1] * b[0] - A[0][1] * b[1]) * inv_det
    grad1 = (-A[1][0] * b[0] + A[0][0] * b[1]) * inv_det
    
    return [grad0, grad1]
```

**Edge Cases and Pitfalls:**
*   **Degeneracy:** The mesh can become degenerate (e.g., vertices becoming colinear) during densification. This is handled by detecting degeneracy and re-meshing, although this rarely occurs in practice [S1].
*   **Gradient Estimation:** The least-squares system for gradient estimation must be solved carefully. If the one-ring neighborhood is degenerate (e.g., all neighbors lie on a line), the matrix $\mathbf{A}_v$ becomes singular. A guard against this is required [G].
*   **Precision:** The method involves multiple floating-point operations. Standard barycentric interpolation is C0, leading to high-frequency artifacts and visible seams in fully opaque meshes [S1]. Clough-Tocher eliminates these by ensuring C1 continuity.

## Operating characteristics
*   **Time Complexity:** The interpolation itself is O(1) per triangle, involving a fixed number of arithmetic operations (dot products, additions, multiplications). The gradient estimation is O(N) per vertex where N is the number of neighbors, but this is typically done as a preprocessing step or in a CUDA kernel during training [S1].
*   **Memory Complexity:** Requires storing 10 control points per triangle (or equivalently, 3 vertex values and 3 vertex gradients). This is slightly more memory than linear interpolation (3 values) but less than storing explicit control points if gradients are derived on the fly.
*   **Measured Performance:** Training generally takes between 1-3 minutes on an RTX A6000 Ada [S1].
*   **Strengths:** Eliminates seam artifacts in fully opaque meshes; provides C1 continuity; differentiable, allowing gradients to flow through vertex topology during training [S1].
*   **Limitations:** Requires per-vertex gradients, which must be estimated or learned; slightly more computationally expensive than linear interpolation.
*   **When to use:** When rendering fully opaque meshes with feature fields (normals, colors, UVs) where visual continuity across triangle edges is critical.
*   **When not to use:** For transparent meshes where seams are less visible or for real-time applications with extremely tight memory constraints where storing gradients is prohibitive.

## Background
The Clough–Tocher scheme was originally developed for C1 interpolation over triangulated domains [S1]. In the context of neural reconstruction, it is used to replace linear barycentric interpolation to eliminate gradient discontinuities that manifest as visible seam artifacts along every edge in fully opaque meshes [S1]. The method is implemented using custom forward and backward CUDA kernels for rasterization and interpolation [S1].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
