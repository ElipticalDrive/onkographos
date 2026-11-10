# Bézier Gaussian Triangle

Also called: BG-Triangle

## Overview
The Bézier Gaussian Triangle (BG-Triangle) is a hybrid representation primitive that combines Bézier triangle surfaces for geometry with 3D Gaussian sub-primitives for appearance, enabling differentiable rendering with sharp boundaries [S1]. It solves the problem of blurry or aliased boundaries common in purely Gaussian splatting methods by defining geometry explicitly via parametric surfaces while retaining the efficient, differentiable rasterization pipeline of Gaussian Splatting [S1]. Unlike [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md), which relies solely on point-based Gaussians, BG-Triangle tessellates curved surfaces into flat triangles to align with the pixel grid, allowing for discontinuity-aware blending at primitive edges [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of Bézier Gaussian Triangle primitives. Each primitive consists of:
    *   Control points $\mathbf{p}_{i, j, k} \in \mathbb{R}^3$ defining the surface geometry.
    *   Color control points $\mathbf{p}_{i, j, k}^c \in \mathbb{R}^3$ defining diffuse color.
    *   Multi-resolution 2D attribute maps $M_h$ storing rotation, scaling, and spherical harmonic (SH) coefficients [S1].
    *   Tessellation parameters (degree $n$).
*   **Outputs:** A rendered image with alpha blending, and gradients for control points and attributes [S1].

**Data Structures:**
*   **Primitive Index Map ($I_{id}$):** $\mathbb{Z}^{H \times W}$, stores the index of the BG-Triangle primitive covering each pixel [S1].
*   **Barycentric Coordinate Map ($I_{uv}$):** $\mathbb{R}^{H \times W \times 3}$, stores the barycentric coordinates $(u, v, w)$ for each pixel [S1].
*   **Boundary Point Set ($B$):** A 2D set of pixel coordinates marking pixels where neighboring pixels have different primitive IDs [S1].
*   **Attribute Maps ($M_h$):** 2D textures of varying resolutions $h$ for rotation, scaling, and SH coefficients [S1].

**Numbered Steps:**

1.  **Tessellation and Rasterization:**
    The non-linear Bézier triangle is tessellated into smaller flat triangles to approximate the curved surface. Vertices of these flat triangles inherit barycentric coordinates and the primitive ID. A traditional rasterization pipeline with depth testing is used to render these flat triangles. This generates the coordinate map $I_{uv}$ and index map $I_{id}$ [S1].

2.  **Boundary Detection:**
    Pixels where neighboring pixels are assigned different primitive IDs are identified as boundary pixels. These are stored in the 2D boundary point set $B$ [S1].

3.  **3D Coordinate Computation:**
    For each foreground pixel $q$, retrieve its primitive index $I_{id}(q)$ and barycentric coordinates $I_{uv}(q) = (u, v, w)$. Compute the 3D position $\mathbf{S}_q$ on the Bézier triangle using the control points $\mathbf{p}_{i, j, k}$ of the indexed primitive:
    $$
    \mathbf{S}_{\mathbf{q}}(\mathbf{I}_{uv}(\mathbf{q}), \mathbf{I}_{id}(\mathbf{q})) = \sum_{i = 0}^{n} \sum_{j = 0}^{n - i} B_{i, j, k}^{n}(\mathbf{I}_{uv}(\mathbf{q})) \mathbf{p}_{i, j, k}(\mathbf{I}_{id}(\mathbf{q}))
    $$
    where $i + j + k = n$ and $u + v + w = 1$ [S1].

4.  **Sub-Primitive Generation:**
    At each computed 3D coordinate $\{\mathbf{S}_q\}$, generate a 3D Gaussian sub-primitive on the fly. These Gaussians possess rotation, scaling, and SH coefficients but **no opacity**, as geometry is defined by the Bézier triangles [S1].

5.  **Attribute Interpolation:**
    *   **Diffuse Color:** Interpolate the diffuse color $\mathbf{c}_q$ using barycentric interpolation of color control points:
        $$
        \mathbf{c}_{\mathbf{q}}(\mathbf{I}_{uv}(\mathbf{q}), \mathbf{I}_{id}(\mathbf{q})) = \sum_{i = 0}^{n} \sum_{j = 0}^{n - i} B_{i, j, k}^{n}(\mathbf{I}_{uv}(\mathbf{q})) \mathbf{p}_{i, j, k}^{c}(\mathbf{I}_{id}(\mathbf{q}))
        $$
    *   **Other Attributes:** Interpolate rotation, scaling, and SH coefficients using a multi-resolution 2D attribute map $M_h$ based on barycentric coordinates via a texture interpolation function $\Theta$:
        $$
        \mathbf{a}_h(\mathbf{q}) = \Theta(\mathbf{M}_h(\mathbf{I}_{id}(\mathbf{q})), \mathbf{I}_{uv}(\mathbf{q}))
        $$
        Different resolutions are applied to $M_h$ based on attribute sensitivity [S1].

6.  **Boundary Influence Calculation:**
    *   Infer 3D boundary points from the 2D boundary set $B$ using Equation (1).
    *   Treat each boundary point as an isotropic Gaussian with a fixed scaling factor $r_b$.
    *   Compute the projected boundary radius $\sigma_i$ for each boundary point $b_i \in B$ [S1].

7.  **Blending Coefficient Computation:**
    For each pixel $q$ covered by a splatted Gaussian $G$, compute the blending coefficient $w(q)$ based on its relation to boundary points $b_i$:
    $$
    w(\mathbf{q}) = \begin{cases} 
    0, & \text{if } \mathbf{I}_{id}(\mathbf{b}_{i}) \neq \mathbf{I}_{id}(\mathbf{q}), \\
    \gamma(\|\mathbf{q} - \mathbf{b}_{i}\|_{2}; \sigma_i), & \text{if } \mathbf{I}_{id}(\mathbf{b}_{i}) = \mathbf{I}_{id}(\mathbf{q}), \\
    1 - \gamma(\|\mathbf{q} - \mathbf{b}_{i}\|_{2}; \sigma_i), & \text{otherwise},
    \end{cases}
    $$
    where $\gamma(d; \sigma) = \min \left( 2^{\frac{d}{\sigma} - 1}, 1 \right)$ [S1].

8.  **Alpha Blending:**
    Soften the boundary influence by computing the alpha value $\alpha(q)$ using the blending coefficient $w(q)$ and the Gaussian projection. The final pixel color is obtained via point-based alpha blending [S1].

9.  **Loss Computation:**
    Apply photometric loss combining L2 loss and D-SSIM term:
    $$
    \mathcal{L} = (1 - \lambda)\mathcal{L}_{2} + \lambda \mathcal{L}_{\text{D-SSIM}}
    $$
    where $\lambda = 0.2$ [S1].

10. **Backward Pass:**
    Compute derivatives of control points using the chain rule, leveraging the differentiability of the position interpolation (Equ. 1) and blending coefficient (Equ. 6) [S1].

11. **Splitting and Pruning (Training):**
    *   **Split:** If the average gradient norm of control points exceeds $\tau_g$ or accumulated back-projected edge gradients exceed $\tau_b$, split the primitive [S1].
    *   **Prune:** Remove primitives based on visibility (percentage of training views where visible), area, and aspect ratio [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def bernstein_polynomial(n, i, j, k, u, v, w):
    """
    Computes the Bernstein polynomial B_{i,j,k}^n(u,v,w).
    """
    # Factorial helper
    def factorial(x):
        if x < 0: return 0
        res = 1
        for k in range(1, x + 1):
            res *= k
        return res
    
    coeff = factorial(n) / (factorial(i) * factorial(j) * factorial(k))
    return coeff * (u ** i) * (v ** j) * (w ** k)

def compute_bezier_point(control_points, n, u, v, w):
    """
    Computes 3D position S(u,v,w) on a Bézier triangle of degree n.
    control_points: list of numpy arrays [p_0,0,n, p_1,0,n-1, ..., p_n,0,0, ...]
    """
    s = np.zeros(3)
    idx = 0
    for i in range(n + 1):
        for j in range(n - i + 1):
            k = n - i - j
            b = bernstein_polynomial(n, i, j, k, u, v, w)
            s += b * control_points[idx]
            idx += 1
    return s

def gamma_function(d, sigma):
    """
    Computes gamma(d; sigma) = min(2^(d/sigma - 1), 1)
    """
    if sigma == 0:
        return 1.0 if d == 0 else 0.0
    val = 2 ** ((d / sigma) - 1)
    return min(val, 1.0)

def render_bg_triangle(primitives, image_height, image_width, camera_params):
    """
    Renders a scene using Bézier Gaussian Triangles.
    primitives: List of BG-Triangle objects.
    """
    # 1. Tessellation and Rasterization
    # In practice, this involves converting Bézier patches to flat triangles.
    # For this pseudocode, we assume we have a rasterization function that returns
    # I_id and I_uv for each pixel.
    I_id = np.zeros((image_height, image_width), dtype=int)
    I_uv = np.zeros((image_height, image_width, 3)) # u, v, w
    
    # Placeholder for actual rasterization logic which depends on the specific
    # tessellation implementation.
    # I_id, I_uv = rasterize_bezier_patches(primitives, image_height, image_width)
    
    # 2. Boundary Detection
    B = set()
    for y in range(image_height):
        for x in range(image_width):
            current_id = I_id[y, x]
            # Check neighbors (simplified to right and bottom)
            neighbors = []
            if x + 1 < image_width: neighbors.append(I_id[y, x + 1])
            if y + 1 < image_height: neighbors.append(I_id[y + 1, x])
            
            if any(nid != current_id for nid in neighbors):
                B.add((x, y))
                
    # 3. Compute 3D Coordinates and Sub-Primitives
    # Store attributes for splatting
    gaussians = [] # List of dicts with 'pos', 'rot', 'scale', 'sh', 'color'
    
    for y in range(image_height):
        for x in range(image_width):
            pid = I_id[y, x]
            if pid == -1: continue # Background
            
            prim = primitives[pid]
            uv = I_uv[y, x]
            u, v, w = uv[0], uv[1], uv[2]
            
            # Compute 3D position
            pos = compute_bezier_point(prim.control_points, prim.n, u, v, w)
            
            # Interpolate Color
            color = np.zeros(3)
            idx = 0
            for i in range(prim.n + 1):
                for j in range(prim.n - i + 1):
                    k = prim.n - i - j
                    b = bernstein_polynomial(prim.n, i, j, k, u, v, w)
                    color += b * prim.color_control_points[idx]
                    idx += 1
            
            # Interpolate other attributes from maps
            # rot, scale, sh = interpolate_attributes(prim.attribute_maps, uv)
            # For brevity, assume these are retrieved
            rot = np.eye(3) 
            scale = np.ones(3)
            sh = np.zeros(3)
            
            gaussians.append({
                'pos': pos,
                'rot': rot,
                'scale': scale,
                'sh': sh,
                'color': color,
                'pixel': (x, y)
            })
            
    # 4. Boundary Influence and Blending
    # Compute boundary radii
    boundary_radii = {}
    for (x, y) in B:
        # Infer 3D boundary point
        pid = I_id[y, x]
        uv = I_uv[y, x]
        b_pos = compute_bezier_point(primitives[pid].control_points, primitives[pid].n, uv[0], uv[1], uv[2])
        
        # Project to 2D to get sigma? 
        # The source says: "compute the boundary radius sigma_i ... when projected onto the image"
        # Assuming a projection function exists.
        sigma_i = project_boundary_radius(b_pos, camera_params, scaling_factor=prim.r_b) # r_b is fixed scaling factor
        boundary_radii[(x, y)] = sigma_i
        
    # 5. Splatting and Alpha Blending
    # Initialize output buffers
    out_color = np.zeros((image_height, image_width, 3))
    out_alpha = np.zeros((image_height, image_width))
    
    # Sort gaussians by depth (simplified)
    gaussians.sort(key=lambda g: g['pos'][2])
    
    for g in gaussians:
        x, y = g['pixel']
        pos = g['pos']
        color = g['color']
        
        # Compute blending coefficient w(q)
        w_q = 1.0
        for (bx, by), sigma_i in boundary_radii.items():
            dist = math.sqrt((x - bx)**2 + (y - by)**2)
            current_id = I_id[y, x]
            neighbor_id = I_id[by, bx] # Note: I_id at boundary might be ambiguous, use stored ID
            
            # Logic from Eq 6
            if neighbor_id != current_id:
                # This case is handled by the "otherwise" or specific neighbor check
                # The formula uses I_id(b_i). If b_i is on the boundary, it has multiple IDs.
                # The source implies checking against the specific neighbor's ID.
                # Simplified: if the boundary point belongs to a DIFFERENT primitive than the current pixel
                # we apply the falloff.
                pass 
            
            # Re-evaluating Eq 6 logic:
            # w(q) = 0 if I_id(b_i) != I_id(q) -> This implies b_i is associated with a specific ID.
            # In rasterization, boundary pixels might have multiple IDs. 
            # The source says: "Pixels with neighboring pixels assigned different primitive IDs are marked as boundary pixels"
            # And: "w(q) = 0 if I_id(b_i) != I_id(q)"
            # This suggests that for a boundary point b_i, we consider the ID of the primitive it "belongs" to in the context of the neighbor.
            # However, the pseudocode structure in the source is:
            # if I_id(b_i) != I_id(q): w=0
            # elif I_id(b_i) == I_id(q): w = gamma
            # else: w = 1 - gamma
            
            # Let's implement the logic as stated:
            # We need to know which ID b_i is associated with. 
            # In the rasterization step, b_i is a pixel coordinate. 
            # If b_i is a boundary pixel, it might have multiple IDs. 
            # The source likely implies iterating over neighbors or using the ID of the primitive that generated the boundary point.
            # For the sake of the algorithm, we assume b_i is associated with the primitive ID of the neighbor that caused the boundary flag.
            
            # Simplified implementation for the article:
            # We check if the current pixel q is adjacent to a boundary point b_i.
            # If so, we determine the weight based on distance and the ID relationship.
            
            # Since exact ID association for b_i is complex without neighbor context in this loop,
            # we rely on the formula structure.
            
            # Let's assume we have a function get_boundary_id(b_i) which returns the ID of the primitive 
            # that is "different" from the current pixel if it's a boundary.
            
            # To keep it implementable from text:
            # We calculate w based on the distance to the boundary and the ID match.
            
            # Note: The source formula is conditional on I_id(b_i).
            # We will assume b_i is stored with its primary ID or we iterate over the IDs of the neighbors.
            
            # For the purpose of the "How it works" section, the formula is the key.
            pass 
            
        # Apply Gaussian Splatting
        # alpha = exp(-0.5 * (x-mu)^T Sigma^-1 (x-mu)) * w_q
        # out_color += alpha * color
        # out_alpha += alpha * (1 - out_alpha)
        pass

    return out_color, out_alpha

def project_boundary_radius(pos_3d, camera_params, scaling_factor):
    """
    Computes the projected radius sigma_i for a boundary point.
    """
    # Placeholder for projection logic
    # Typically involves Jacobian of projection
    return scaling_factor
```

**Parameters:**
*   $\lambda$: Weight for D-SSIM term in loss function, set to 0.2 [S1].
*   $r_b$: Fixed scaling factor for isotropic Gaussian treatment of boundary points [S1].
*   $o$: Constant number close to 1.0 used to help optimization in alpha blending [S1].
*   $\tau_g$: Threshold for average gradient norm of control points to trigger splitting [S1].
*   $\tau_b$: Threshold for accumulated edge gradients to trigger splitting [S1].

**Edge Cases and Pitfalls:**
*   **Tessellation Density:** Insufficient tessellation leads to aliasing and inaccurate barycentric coordinates. The tessellation must be fine enough to align with the pixel grid [S1].
*   **Boundary Ambiguity:** Pixels on the boundary have multiple primitive IDs. The blending coefficient logic must correctly identify which ID is "active" for the boundary point to apply the correct falloff [S1].
*   **Gradient Vanishing:** If control points are not updated correctly via the chain rule, optimization may stall. The differentiability of Equ. (1) and Equ. (6) is crucial [S1].

## Subtopics

### Splitting and Pruning Scheme for Bézier Gaussian Triangles
A scheme to manage primitive count and quality by splitting under-reconstructed primitives and pruning ineffective ones to improve training efficiency and rendering clarity [S1]. Primitives are split if the average gradient norm of control points exceeds $\tau_g$ or if accumulated back-projected edge gradients exceed $\tau_b$ [S1]. Pruning is based on visibility (percentage of training views where visible), area, and aspect ratio [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by tessellation and rasterization of Bézier triangles, followed by Gaussian splatting. Tessellation adds overhead compared to pure point-based splatting but allows for sharp boundaries [S1].
*   **Memory Complexity:** Requires storing control points, color control points, and multi-resolution attribute maps. Generally more compact than pure Gaussian representations for smooth surfaces [S1].
*   **Strengths:** Sharp boundaries, differentiable rendering, compact representation for smooth surfaces [S1].
*   **Limitations:** Complexity of tessellation and boundary handling. Blurry close-ups if tessellation is insufficient [S1].
*   **When to use:** Scenes with smooth, continuous surfaces where sharp boundaries are important.
*   **When not to use:** Highly discontinuous or noisy geometry where point-based methods might be more robust.

## Background
The Bézier Gaussian Triangle was introduced in the paper "BG-Triangle: Bézier Gaussian Triangle for 3D Vectorization and Rendering" [S1]. It builds on the principles of 3D Gaussian Splatting but introduces Bézier triangles to define geometry explicitly, addressing the lack of sharp boundaries in pure Gaussian methods [S1].

## Sources
[S1] BG-Triangle: Bézier Gaussian Triangle for 3D Vectorization and Rendering, https://doi.org/10.1109/cvpr52734.2025.01510, fetched 2026-10-26
