# HexPlane


## Overview
HexPlane is an explicit representation for dynamic 3D scenes that decomposes a 4D spacetime grid into six feature planes spanning each pair of coordinate axes (e.g., XY, ZT) [S1]. It computes features for points in spacetime by projecting the point onto each feature plane, aggregating the six resulting feature vectors, and passing the fused vector to a tiny MLP to predict color [S1]. It sets itself apart from implicit NeRF-based methods by avoiding large MLP evaluations for feature extraction, achieving comparable or better synthesis quality for dynamic novel view synthesis with over 100× acceleration [S1]. Unlike deformation-based methods, it does not rely on deformation fields, making it robust to topology changes but potentially less effective for extremely sparse monocular observations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed and timestamped images of a dynamic scene.
*   **Output:** A parametric function mapping a 3D position $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\mathbf{d} \in \mathbb{R}^3$ to volume density $\sigma$ and RGB color $\mathbf{c}$, enabling differentiable volumetric rendering.

**Data Structures:**
The representation consists of two HexPlanes (one for opacity, one for appearance) and a tiny MLP.
1.  **HexPlane:** A set of six learned feature planes. Each plane is a 2D tensor. The planes are grouped into three pairs:
    *   Pair 1: $M^{XY} \in \mathbb{R}^{X \times Y}$ and $M^{ZT} \in \mathbb{R}^{Z \times T}$
    *   Pair 2: $M^{XZ} \in \mathbb{R}^{X \times Z}$ and $M^{YT} \in \mathbb{R}^{Y \times T}$
    *   Pair 3: $M^{YZ} \in \mathbb{R}^{Y \times Z}$ and $M^{XT} \in \mathbb{R}^{X \times T}$
    *   Note: $X, Y, Z$ are spatial grid resolutions; $T$ is temporal resolution.
2.  **VRF (Vector-Response-Fusion) Matrices:** Three matrices $V^{R_1 F}, V^{R_2 F}, V^{R_3 F}$ where $R_i$ is the rank (number of components) for pair $i$, and $F$ is the feature dimension.
3.  **Tiny MLP:** A small neural network taking appearance features and view direction to regress RGB.
4.  **Emptiness Voxel:** A tiny 3D voxel grid indicating empty regions, computed by taking the maximum opacity across time steps.

**Parameters:**
*   $N$: Spatial resolution (e.g., 512).
*   $T$: Temporal resolution (e.g., 32 or 300).
*   $F$: Feature size (e.g., 3 for RGB, or higher for SH).
*   $R_1, R_2, R_3$: Basis numbers (ranks). Typical values: 24, 48, 96.
*   Coordinate System: Normalized Device Coordinates (NDC) for bounded scenes, or Spherical Coordinates for unbounded scenes.

**Formulas:**
The HexPlane query function $D(x, y, z, t)$ maps a 4D point to an $F$-dimensional feature vector:
$$
D(x, y, z, t) = (\mathbf{P}_{xy\bullet}^{XYR1} \odot \mathbf{P}_{xt\bullet}^{ZTR1}) \mathbf{V}^{R1F} + (\mathbf{P}_{xz\bullet}^{XZR2} \odot \mathbf{P}_{yt\bullet}^{YTR2}) \mathbf{V}^{R2F} + (\mathbf{P}_{yz\bullet}^{YZR3} \odot \mathbf{P}_{xt\bullet}^{XTR3}) \mathbf{V}^{R3F}
$$
where $\odot$ is element-wise product, $\mathbf{P}$ tensors stack planes, and subscripts denote bilinear interpolation.

**Numbered Steps:**
1.  **Initialization:** Initialize six feature planes for each HexPlane (opacity and appearance) and VRF matrices. Initialize emptiness voxel.
2.  **Ray Sampling:** For each pixel, sample points $\mathbf{x}_i = (x, y, z, t)$ along the ray. If using NDC, coordinates are bounded. If using Spherical Coordinates, re-parameterize $(x, y, z)$ to $(\theta, \phi, r)$ and sample $r$ linearly between 0 and 1.
3.  **Emptiness Check:** If the point lies in an empty region (checked via emptiness voxel), skip rendering.
4.  **Opacity Query:** Query the opacity HexPlane at $(x, y, z, t)$ to get opacity $\sigma$.
5.  **Appearance Query:** If $\sigma$ is high (threshold defined by implementation, e.g., [G] $\sigma > 0.01$), query the appearance HexPlane at $(x, y, z, t)$ to get appearance feature $\mathbf{f}_{app}$.
6.  **Color Regression:** Pass $\mathbf{f}_{app}$ and view direction $\mathbf{d}$ to the tiny MLP to regress RGB color $\mathbf{c}$.
7.  **Volumetric Rendering:** Compute pixel color $C(\mathbf{r})$ using standard volume rendering:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
8.  **Loss Calculation:** Compute photometric loss $\mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \|C(r) - \hat{C}(r)\|_2^2 + \lambda_{reg} L_{reg}$.
9.  **Regularization:** Apply Total Variational (TV) loss on all feature planes to enforce spatial-temporal continuity. Apply depth smooth loss to reduce artifacts.
10. **Optimization:** Update HexPlane planes, VRF matrices, and MLP weights using Adam optimizer.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def hexplane_query(plane_xy, plane_zt, plane_xz, plane_yt, plane_yz, plane_xt, 
                   vrf, x, y, z, t):
    """
    Query a single HexPlane term.
    plane_* : 2D numpy arrays (tensors) of shape (dim1, dim2)
    vrf : 2D numpy array of shape (R, F)
    x, y, z, t : float coordinates
    """
    # Bilinear interpolation helper
    def bilinear_interp(plane, u, v):
        # u, v are normalized coordinates [0, 1]
        # Get grid dimensions
        H, W = plane.shape
        # Map to grid indices
        u_idx = u * (W - 1)
        v_idx = v * (H - 1)
        
        # Floor and ceil
        u0 = int(np.floor(u_idx))
        v0 = int(np.floor(v_idx))
        u1 = u0 + 1
        v1 = v0 + 1
        
        # Clamp
        u0 = np.clip(u0, 0, W - 2)
        u1 = np.clip(u1, 0, W - 1)
        v0 = np.clip(v0, 0, H - 2)
        v1 = np.clip(v1, 0, H - 1)
        
        # Interpolation weights
        su = u_idx - u0
        sv = v_idx - v0
        
        # Bilinear interpolation
        val = (plane[v0, u0] * (1 - su) * (1 - sv) +
               plane[v0, u1] * (1 - su) * sv +
               plane[v1, u0] * su * (1 - sv) +
               plane[v1, u1] * su * sv)
        return val

    # Extract features from planes
    # Term 1: XY and ZT
    f_xy = bilinear_interp(plane_xy, x, y)
    f_zt = bilinear_interp(plane_zt, z, t)
    # Element-wise product
    f_term1 = f_xy * f_zt
    
    # Term 2: XZ and YT
    f_xz = bilinear_interp(plane_xz, x, z)
    f_yt = bilinear_interp(plane_yt, y, t)
    f_term2 = f_xz * f_yt
    
    # Term 3: YZ and XT
    f_yz = bilinear_interp(plane_yz, y, z)
    f_xt = bilinear_interp(plane_xt, x, t)
    f_term3 = f_yz * f_xt
    
    # Concatenate terms
    fused = np.concatenate([f_term1, f_term2, f_term3])
    
    # Multiply by VRF
    # fused shape: (3*R,) -> split into 3 chunks of R
    # vrf shape: (3*R, F)
    # Actually, vrf is stacked: [V1, V2, V3]
    # So we do matrix multiplication
    result = fused @ vrf.T # Result shape: (F,)
    
    return result

def render_hexplane(rays, opacity_plane, appearance_plane, mlp, emptiness_voxel, 
                    grid_res_x, grid_res_y, grid_res_z, grid_res_t):
    """
    Render a batch of rays.
    rays: list of (origin, direction, t) tuples
    """
    rendered_colors = []
    
    for origin, direction, t in rays:
        # Sample points along ray
        # Assume N samples
        points = sample_points(origin, direction, N=128) # Implementation specific
        
        alpha_sum = 0.0
        color = np.zeros(3)
        
        for point in points:
            x, y, z = point[:3]
            
            # Check emptiness
            if is_empty(emptiness_voxel, x, y, z):
                continue
                
            # Normalize coordinates to [0, 1] for grid lookup
            # Depends on coordinate system (NDC or Spherical)
            nx, ny, nz, nt = normalize_coords(x, y, z, t, grid_res_x, grid_res_y, grid_res_z, grid_res_t)
            
            # Query Opacity
            # Opacity HexPlane uses specific planes
            sigma = hexplane_query(opacity_plane.xy, opacity_plane.zt, 
                                   opacity_plane.xz, opacity_plane.yt, 
                                   opacity_plane.yz, opacity_plane.xt, 
                                   opacity_plane.vrf, nx, ny, nz, nt)
            # Sigma is a vector, take first component or mean? 
            # Source says "opacity queried directly from one HexPlane"
            # Assuming scalar output or first dim
            sigma = sigma[0] if len(sigma) > 0 else 0.0
            
            alpha = 1.0 - np.exp(-sigma * delta) # delta is step size
            
            weight = (1.0 - alpha_sum) * alpha
            alpha_sum += weight
            
            if weight > 1e-4:
                # Query Appearance
                appearance_feat = hexplane_query(appearance_plane.xy, appearance_plane.zt,
                                                 appearance_plane.xz, appearance_plane.yt,
                                                 appearance_plane.yz, appearance_plane.xt,
                                                 appearance_plane.vrf, nx, ny, nz, nt)
                
                # Regress Color
                rgb = mlp(appearance_feat, direction)
                color += weight * rgb
                
        rendered_colors.append(color)
        
    return rendered_colors
```

**Edge Cases and Pitfalls:**
*   **Out-of-Boundaries:** HexPlane is limited to bounded scenes in NDC. Grid sampling fails for out-of-boundary points, leading to artifacts [S1]. Use Spherical Coordinates for unbounded scenes [S1].
*   **Fast Motion/Sparse Observations:** Degraded results when objects move too fast or observations are too sparse, as the basis-sharing mechanism is less effective than deformation fields in these cases [S1].
*   **Artifacts:** Explicit representations are prone to artifacts and color jittering, requiring strong regularization (TV loss) [S1].
*   **Overfitting:** Unsuitable large $R$ may lead to overfitting [S1].

## Operating characteristics
*   **Time Complexity:** Querying a HexPlane requires six bilinear interpolations and one vector-matrix product per term, making it significantly faster than implicit MLP evaluations [S1].
*   **Memory Complexity:** Memory usage is reduced from $O(N^3TF)$ to $O(N^2R + NTR + RF)$, where $N$ is spatial resolution, $T$ is temporal resolution, $R$ is rank, and $F$ is feature dimension [S1].
*   **Measured Performance:** On the Plenoptic Video dataset, HexPlane achieves 29.470 PSNR in 12 GPU hours (112x speedup over DyNeRF) [S1]. HexPlane† (100k iterations) achieves 29.263 PSNR in 2 GPU hours (672x speedup) [S1]. On D-NeRF dataset, it achieves 31.04 PSNR in 11 minutes [S1].
*   **Model Size:** 200MB [S1].
*   **Strengths:** High efficiency, explicit representation allows fast rendering, robust to topology changes (no deformation fields), good quality for dynamic scenes.
*   **Limitations:** Limited to bounded scenes in NDC, artifacts/jittering, struggles with extremely sparse observations or very fast motion compared to deformation-based methods.

## Background
HexPlane was introduced by Ang Cao and Justin Johnson from the University of Michigan, Ann Arbor, in 2023 [S1]. It builds on the idea of factored representations for static scenes (like TensoRF) and extends it to 4D spacetime by using spatial-temporal planes to capture entangled space-time dynamics [S1]. It addresses the challenges of memory usage and sparse observations in dynamic 3D scene reconstruction [S1].

## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
