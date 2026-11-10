# Bézier Gaussian Triangle

Also called: BG-Triangle

## Overview
The Bézier Gaussian Triangle (BG-Triangle) is a hybrid 3D primitive representation that combines Bézier triangle surfaces for explicit geometry with 3D Gaussian sub-primitives for appearance, enabling differentiable rendering with sharp boundaries [S1]. It solves the problem of blurry or aliased edges common in pure Gaussian splatting by defining the object's silhouette and surface topology via parametric Bézier patches, while using Gaussians only for shading and color [S1]. Unlike standard 3D Gaussian Splatting, which relies on the overlap of many opaque Gaussians to form shapes, BG-Triangle maintains distinct, sharp boundaries between primitives, making it particularly suitable for vectorization and compact scene representation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of Bézier Gaussian Triangle primitives. Each primitive is defined by a degree $n$, a set of 3D control points $\mathbf{p}_{i,j,k}$, color control points $\mathbf{p}_{i,j,k}^c$, and multi-resolution attribute maps $M_h$ for rotation, scaling, and Spherical Harmonic (SH) coefficients.
*   **Output:** A rendered 2D image with photometric accuracy and sharp geometric boundaries.

**Data Structures:**
*   **Primitive Index Map ($I_{id}$):** $\mathbb{Z}^{H \times W}$, storing the index of the primitive covering each pixel.
*   **Barycentric Coordinate Map ($I_{uv}$):** $\mathbb{R}^{H \times W \times 3}$, storing the barycentric coordinates $(u, v, w)$ for each pixel relative to its covering primitive.
*   **Boundary Point Set ($B$):** A 2D set of pixel coordinates identified as boundary pixels (pixels with neighbors having different $I_{id}$ values).
*   **Attribute Maps ($M_h$):** 2D textures storing rotation, scaling, and SH coefficients for each primitive, sampled based on barycentric coordinates.

**Numbered Steps:**

1.  **Tessellation and Rasterization:**
    The non-linear Bézier triangle surface is tessellated into smaller flat triangles to align with the pixel grid. The vertices of these flat triangles inherit the primitive ID and barycentric coordinates. A traditional rasterization pipeline with depth testing is used to render these flat triangles, generating the coordinate map $I_{uv}$ and index map $I_{id}$ [S1].

2.  **Boundary Detection:**
    Pixels are marked as boundary pixels if they have neighboring pixels assigned to different primitive IDs. These pixels are stored in the 2D boundary point set $B$ [S1].

3.  **3D Coordinate Computation:**
    For each foreground pixel $q$, the 3D position $\mathbf{S}_q$ on the Bézier triangle is computed using the retrieved primitive index $I_{id}(q)$ and barycentric coordinates $I_{uv}(q)$:
    $$
    \mathbf{S}_{\mathbf{q}}(\mathbf{I}_{uv}(\mathbf{q}), \mathbf{I}_{id}(\mathbf{q})) = \sum_{i = 0}^{n} \sum_{j = 0}^{n - i} B_{i, j, k}^{n}(\mathbf{I}_{uv}(\mathbf{q})) \mathbf{p}_{i, j, k}(\mathbf{I}_{id}(\mathbf{q}))
    $$
    where $B_{i, j, k}^{n}$ is the Bernstein polynomial [S1].

4.  **Sub-Primitive Generation:**
    At each computed 3D coordinate $\{\mathbf{S}_q\}$, a 3D Gaussian sub-primitive is generated on the fly. These Gaussians possess rotation, scaling, and SH coefficients but **no opacity**, as the geometry is defined by the Bézier triangles [S1].

5.  **Attribute Interpolation:**
    *   **Diffuse Color:** Interpolated using barycentric interpolation of color control points $\mathbf{p}_{i, j, k}^c$:
        $$
        \mathbf{c}_{\mathbf{q}}(\mathbf{I}_{uv}(\mathbf{q}), \mathbf{I}_{id}(\mathbf{q})) = \sum_{i = 0}^{n} \sum_{j = 0}^{n - i} B_{i, j, k}^{n}(\mathbf{I}_{uv}(\mathbf{q})) \mathbf{p}_{i, j, k}^{c}(\mathbf{I}_{id}(\mathbf{q}))
        $$
    *   **Other Attributes:** Rotation, scaling, and SH coefficients are interpolated using a multi-resolution 2D attribute map $M_h$ based on barycentric coordinates via a texture interpolation function $\Theta$:
        $$
        \mathbf{a}_h(\mathbf{q}) = \Theta(\mathbf{M}_h(\mathbf{I}_{id}(\mathbf{q})), \mathbf{I}_{uv}(\mathbf{q}))
        $$
        where $h \in \{\text{rotation, scaling, SH coefficients}\}$ [S1].

6.  **Boundary Softening (Discontinuity-Aware Rendering):**
    To handle sharp boundaries without artifacts, the influence of boundary points is softened:
    *   Each boundary point $b_i \in B$ is treated as an isotropic Gaussian with a fixed scaling factor $r_b$.
    *   The projected boundary radius $\sigma_i$ is computed for each boundary Gaussian.
    *   The blending coefficient $w(q)$ for a pixel $q$ covered by a splatted Gaussian $G$ is determined by its relation to boundary points $b_i$:
        $$
        w(\mathbf{q}) = \begin{cases} 
        0, & \text{if } \mathbf{I}_{id}(\mathbf{b}_{i}) \neq \mathbf{I}_{id}(\mathbf{q}), \\
        \gamma(\|\mathbf{q} - \mathbf{b}_{i}\|_{2}; \sigma_i), & \text{if } \mathbf{I}_{id}(\mathbf{b}_{i}) = \mathbf{I}_{id}(\mathbf{q}), \\
        1 - \gamma(\|\mathbf{q} - \mathbf{b}_{i}\|_{2}; \sigma_i), & \text{otherwise},
        \end{cases}
        $$
        where $\gamma(d; \sigma) = \min \left( 2^{\frac{d}{\sigma} - 1}, 1 \right)$ [S1].
    *   The alpha value $\alpha(q)$ is computed using $w(q)$ and the Gaussian projection, where $o$ is a constant close to 1.0 to aid optimization [S1].

7.  **Rendering:**
    Point-based alpha blending is applied to obtain the final pixel color [S1].

8.  **Training (Backward Pass):**
    Derivatives of the control points are computed using the chain rule, leveraging the differentiability of the coordinate interpolation (Equ. 1) and the blending weight (Equ. 6) [S1]. The photometric loss combines L2 loss and D-SSIM:
    $$
    \mathcal{L} = (1 - \lambda)\mathcal{L}_{2} + \lambda \mathcal{L}_{\text{D-SSIM}}
    $$
    where $\lambda = 0.2$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def bernstein_polynomial(i, j, k, n, u, v, w):
    """
    Computes the Bernstein polynomial B_{i,j,k}^n(u,v,w).
    """
    # Factorial helper
    def factorial(x):
        return math.factorial(x)
    
    coeff = factorial(n) / (factorial(i) * factorial(j) * factorial(k))
    return coeff * (u ** i) * (v ** j) * (w ** k)

def compute_bezier_point(control_points, n, u, v, w):
    """
    Computes the 3D position S(u,v,w) on a Bézier triangle.
    control_points: List of 3D vectors p_{i,j,k}.
    n: Degree of the Bézier triangle.
    """
    position = np.zeros(3)
    for i in range(n + 1):
        for j in range(n - i + 1):
            k = n - i - j
            # Retrieve control point p_{i,j,k}
            p = control_points[i][j][k]
            b = bernstein_polynomial(i, j, k, n, u, v, w)
            position += b * p
    return position

def gamma_function(d, sigma):
    """
    Computes gamma(d; sigma) = min(2^(d/sigma - 1), 1).
    """
    if sigma == 0:
        return 1.0 # Fallback, though sigma should be > 0
    val = 2 ** ((d / sigma) - 1)
    return min(val, 1.0)

def compute_blending_weight(q_pos, boundary_points, q_primitive_id, boundary_primitive_ids, sigma_i):
    """
    Computes the blending coefficient w(q) for pixel q.
    """
    w = 0.0
    # Note: In practice, this iterates over relevant boundary points.
    # For simplicity, we assume a single dominant boundary influence or sum them.
    # The source defines w(q) based on relation to b_i.
    # We iterate through boundary points to find the relevant one.
    # In a real implementation, spatial indexing would be used.
    
    for b_i, b_id in zip(boundary_points, boundary_primitive_ids):
        dist = np.linalg.norm(q_pos - b_i)
        
        if b_id != q_primitive_id:
            # Case 1: Different primitive ID
            w_component = 0.0
        elif b_id == q_primitive_id:
            # Case 2: Same primitive ID
            w_component = gamma_function(dist, sigma_i)
        else:
            # Case 3: Otherwise (handled by the 'otherwise' clause in source)
            # This case usually implies checking against other boundaries.
            # The source formula is conditional on b_i.
            # If we are summing contributions, we need to be careful.
            # The source implies w(q) is determined by the specific boundary point b_i
            # that defines the edge.
            w_component = 1.0 - gamma_function(dist, sigma_i)
            
        # The source formula is singular for a specific b_i. 
        # In a rasterization context, w(q) is likely the product or max of influences.
        # However, the text says "determine the blending coefficient w(q)... based on its relation to boundary points b_i".
        # We will assume the standard implementation sums or selects the max influence.
        # Given the complexity, we stick to the direct formula application for a single boundary context.
        # For the purpose of this article, we return the value for the specific b_i.
        return w_component
        
    return w

def render_bg_triangle(primitives, image_width, image_height, camera_params):
    """
    Main rendering loop for BG-Triangle.
    """
    # 1. Tessellation and Rasterization (Simplified)
    # In reality, this involves GPU tessellation. Here we assume we have I_id and I_uv.
    I_id = np.zeros((image_height, image_width), dtype=int)
    I_uv = np.zeros((image_height, image_width, 3))
    B = [] # Boundary points
    
    # Placeholder for rasterization step
    # ... Rasterize flat triangles ...
    # ... Detect boundaries ...
    
    # 2. Initialize Output Image
    final_color = np.zeros((image_height, image_width, 3))
    
    # 3. Forward Pass for each pixel
    for y in range(image_height):
        for x in range(image_width):
            q = (x, y)
            q_pos_2d = np.array([x, y])
            
            prim_id = I_id[y, x]
            uv = I_uv[y, x]
            u, v, w = uv[0], uv[1], uv[2]
            
            # Get primitive data
            prim = primitives[prim_id]
            control_points = prim['control_points']
            color_control_points = prim['color_control_points']
            attr_maps = prim['attr_maps']
            
            # Compute 3D Position
            S_q = compute_bezier_point(control_points, prim['degree'], u, v, w)
            
            # Interpolate Attributes
            # Color
            c_q = compute_bezier_point(color_control_points, prim['degree'], u, v, w)
            
            # Other attributes (Rotation, Scaling, SH) via Theta function
            # Theta is a 2D texture lookup. We simulate it.
            rotation = attr_maps['rotation'].sample(u, v)
            scaling = attr_maps['scaling'].sample(u, v)
            sh_coeffs = attr_maps['sh'].sample(u, v)
            
            # Project Gaussian to 2D (Simplified)
            # Compute 2D covariance Sigma and mean mu from S_q, rotation, scaling, camera
            mu_2d, sigma_2d = project_gaussian(S_q, rotation, scaling, camera_params)
            
            # Compute Alpha and Blending
            # Check boundaries
            w_q = 1.0
            # In a full implementation, check if q is near a boundary in B
            # For simplicity, assume no boundary softening if not near B
            # If near B, compute w(q) using boundary points
            
            # Alpha blending contribution
            # Gaussian PDF
            dist_sq = np.dot((q_pos_2d - mu_2d), np.linalg.inv(sigma_2d) @ (q_pos_2d - mu_2d))
            gaussian_val = np.exp(-0.5 * dist_sq)
            
            # Apply boundary weight if applicable
            # alpha = gaussian_val * w_q * o (o ~ 1.0)
            alpha = gaussian_val * w_q
            
            # Composite color (simplified single primitive)
            final_color[y, x] = c_q * alpha
            
    return final_color
```

**Parameters:**
*   $\lambda$: Weight for D-SSIM term in loss function, set to 0.2 [S1].
*   $r_b$: Fixed scaling factor for isotropic Gaussian treatment of boundary points [S1].
*   $o$: Constant number close to 1.0 used to help optimization in alpha blending [S1].

**Edge Cases and Pitfalls:**
*   **Tessellation Density:** The tessellation of the Bézier triangle must be fine enough to approximate the curved surface without introducing visible artifacts, but coarse enough to maintain performance. [S1]
*   **Boundary Artifacts:** Incorrect handling of boundary pixels can lead to gaps or overlaps between primitives. The discontinuity-aware rendering is crucial for sharp edges. [S1]
*   **Gradient Computation:** The differentiability of the Bézier surface and the blending function is essential for training. Numerical instability in the Bernstein polynomials or gamma function should be avoided. [S1]

## Subtopics

### Splitting and Pruning Scheme for Bézier Gaussian Triangles
A scheme to manage primitive count and quality by splitting under-reconstructed primitives and pruning ineffective ones to improve training efficiency and rendering clarity [S1].
*   **Splitting Criteria:** Primitives are split if: (1) the average gradient norm of their control points exceeds threshold $\tau_g$, indicating under-reconstructed geometry; or (2) the accumulated edge gradients, back-projected from edge detection on training images, exceed threshold $\tau_b$ [S1].
*   **Pruning Criteria:** Primitives are pruned based on: (1) visibility (percentage of training views in which the primitive is visible); (2) area; and (3) aspect ratio [S1].
*   **Parameters:** $\tau_g$ (threshold for average gradient norm), $\tau_b$ (threshold for accumulated edge gradients) [S1].

## Operating characteristics
*   **Time Complexity:** Rendering involves tessellation, rasterization, and per-pixel Gaussian splatting. The tessellation step is offline or pre-computed. The rasterization and splatting are similar in complexity to standard 3D Gaussian Splatting, but with additional overhead for boundary detection and attribute interpolation from Bézier control points [S1].
*   **Memory Complexity:** Memory usage is dominated by the storage of control points, color control points, and multi-resolution attribute maps for each primitive. This is generally more compact than storing millions of individual Gaussians, as the geometry is defined by a smaller number of control points [S1].
*   **Strengths:** Sharp boundaries, differentiable rendering, compact representation, suitability for vectorization [S1].
*   **Limitations:** Complexity of implementation due to hybrid nature (Bézier + Gaussian), sensitivity to tessellation density, potential artifacts at boundaries if not handled correctly [S1].
*   **When to Use:** When sharp geometric boundaries are required, or when a compact representation of complex surfaces is needed. [S1]
*   **When Not to Use:** For scenes with highly complex, non-parametric geometry that cannot be well-approximated by Bézier triangles, or when real-time performance is critical and the overhead of tessellation and boundary handling is unacceptable [S1].

## Background
The Bézier Gaussian Triangle (BG-Triangle) was introduced in the paper "BG-Triangle: Bézier Gaussian Triangle for 3D Vectorization and Rendering" [S1]. It builds upon the principles of 3D Gaussian Splatting [S1] and Bézier surfaces, aiming to combine the differentiable rendering capabilities of Gaussians with the explicit geometric control of Bézier patches. The method addresses the limitations of pure Gaussian splatting in representing sharp edges and compact geometry [S1].

## Sources
[S1] BG-Triangle: Bézier Gaussian Triangle for 3D Vectorization and Rendering, https://doi.org/10.1109/cvpr52734.2025.01510, fetched 2026-10-26
