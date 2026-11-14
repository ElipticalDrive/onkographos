# Gaussian Splatting

Also called: 3D GS, 3D Gaussian Splatting (3DGS)

## Overview
Gaussian Splatting (3DGS) is a method for 3D scene representation and novel view synthesis that models a scene as an explicit collection of learnable 3D Gaussian ellipsoids [S1, S4, S5]. It solves the computational bottleneck of implicit neural radiance fields (NeRFs) by replacing volume query-based ray marching with differentiable object-order rasterization [S1][S18, S30, unofficial]. Unlike NeRFs, which require dense sampling along rays, 3DGS projects 3D Gaussians into 2D elliptical footprints (splats) on the image plane and composites them via alpha blending [S4][S11, unofficial][S15]. This approach enables real-time rendering speeds while maintaining high visual fidelity [S1, S5].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $N_{views}$ posed input images $\{I_i\}$ with known camera intrinsics $K$ and extrinsics (pose matrices $W_i$) [S1, S5].
*   A sparse colored point cloud obtained via Structure-from-Motion (SfM) (e.g., COLMAP) [S1, S14].

**Outputs:**
*   A set of $M$ 3D Gaussians $\mathcal{G} = \{g_i\}_{i=1}^M$, where each Gaussian $g_i$ is defined by parameters optimized to minimize the difference between rendered and input images [S1, S5].
*   Rendered novel views of the scene [S1].

### Data Structures
Each Gaussian $g_i$ is defined by the following parameters [S1, S5][S7, unofficial]:

1.  **Mean (Position):** $\boldsymbol{\mu}_i \in \mathbb{R}^3$. The center of the Gaussian in world coordinates [S1].
2.  **Covariance:** $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$. A symmetric positive-definite matrix describing the shape, size, and orientation [S1, S5]. To ensure positive definiteness during optimization, $\boldsymbol{\Sigma}_i$ is parameterized by a rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$ (normalized to unit length) and a scale vector $\mathbf{s}_i \in \mathbb{R}^3$ [S5][S7, unofficial]:
    $$ \boldsymbol{\Sigma}_i = \mathbf{R}(\mathbf{q}_i) \mathbf{S}(\mathbf{s}_i) \mathbf{S}(\mathbf{s}_i)^\top \mathbf{R}(\mathbf{q}_i)^\top $$
    where $\mathbf{R}$ is the rotation matrix derived from $\mathbf{q}_i$ and $\mathbf{S} = \text{diag}(\mathbf{s}_i)$ [S5][S24, unofficial].
3.  **Opacity:** $\alpha_i \in [0, 1]$. A scalar representing the peak opacity of the Gaussian [S1, S5].
4.  **Color:** Represented by spherical harmonics (SH) coefficients $\mathbf{c}_i \in \mathbb{R}^{3 \times (k+1)^2}$, where $k$ is the band degree [S5][S7, unofficial]. Band 0 (DC) is typically used for constant color, while higher bands capture view-dependent appearance [S7, unofficial]. The color for a viewing direction $\mathbf{v}$ is computed as:
    $$ \mathbf{c}_i(\mathbf{v}) = \sum_{l=0}^{k} \sum_{m=-l}^{l} Y_l^m(\mathbf{v}) \cdot \mathbf{c}_{i, l, m} $$
    where $Y_l^m$ are the SH basis functions [S7, unofficial]. For band 0, $Y_0^0 = \frac{1}{2\sqrt{\pi}} \approx 0.282$, and the RGB color is $\text{clamp}(0.5 + C_0 \cdot f_{dc}, 0, 1)$ [S7, unofficial].

### Rendering Pipeline
The rendering process consists of the following steps [S1, S5, S15]:

1.  **Frustum Culling:** Identify Gaussians outside the camera's view frustum and exclude them [S5].
2.  **3D-to-2D Projection (Splatting):** Project each 3D Gaussian into the 2D image plane [S15].
    *   Transform the mean to camera coordinates: $\boldsymbol{\mu}'_i = \mathbf{W}_i \boldsymbol{\mu}_i$ [S5].
    *   Approximate the perspective projection using a local affine transformation. Let $\mathbf{J} \in \mathbb{R}^{2 \times 3}$ be the Jacobian of the projective transformation at $\boldsymbol{\mu}'_i$ [S1, S5]. The 2D covariance matrix $\boldsymbol{\Sigma}'_i$ is computed as:
        $$ \boldsymbol{\Sigma}'_i = \mathbf{J} \mathbf{W}_i \boldsymbol{\Sigma}_i \mathbf{W}_i^\top \mathbf{J}^\top $$
        The 2D mean is $\boldsymbol{\mu}^{2D}_i = \text{proj}(\boldsymbol{\mu}'_i)$ [S5][S24, unofficial].
    *   The opacity and color are preserved: $\tilde{\alpha}_i = \alpha_i$ and $\tilde{\mathbf{c}}_i = \mathbf{c}_i$ [S1].
3.  **Tile-Based Sorting:**
    *   Divide the image into non-overlapping tiles (typically $16 \times 16$ pixels) [S5, S15].
    *   Assign each 2D Gaussian to the tiles it intersects [S5].
    *   Sort the Gaussians within each tile by depth (distance from camera) using a GPU-optimized radix sort [S5, S15].
4.  **Alpha Compositing:**
    *   For each pixel in a tile, iterate over the sorted Gaussians covering that pixel [S5].
    *   Compute the 2D Gaussian value at pixel coordinate $\mathbf{x}$:
        $$ \mathcal{G}^{2D}_i(\mathbf{x}) = \exp\left(-\frac{1}{2} (\mathbf{x} - \boldsymbol{\mu}^{2D}_i)^\top (\boldsymbol{\Sigma}'_i)^{-1} (\mathbf{x} - \boldsymbol{\mu}^{2D}_i)\right) $$
    *   Compute the effective opacity $\alpha'_i(\mathbf{x}) = \alpha_i \cdot \mathcal{G}^{2D}_i(\mathbf{x})$ [S5][S18, unofficial].
    *   Accumulate color using front-to-back alpha blending [S5, S15]:
        $$ C_{\mathbf{x}} = \sum_{i \in \mathcal{N}} \mathbf{c}_i \alpha'_i(\mathbf{x}) \prod_{j=1}^{i-1} (1 - \alpha'_j(\mathbf{x})) $$
        where $\mathcal{N}$ is the set of sorted Gaussians for the pixel, and the product term represents the accumulated transmittance $T_i$ [S5, S15][S21, unofficial].

### Training
*   **Initialization:** Gaussians are initialized at the positions of the SfM point cloud [S1, S14]. Opacity is initialized to a small constant (e.g., 0.1) [S32, unofficial]. Color is initialized from the average pixel colors of visible images [S32, unofficial].
*   **Optimization:** Parameters ($\boldsymbol{\mu}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i$) are optimized using gradient descent (e.g., Adam) to minimize a loss function [S1, S5]:
    $$ \mathcal{L} = (1 - \lambda) \mathcal{L}_{L1} + \lambda \mathcal{L}_{D-SSIM} $$
    where $\mathcal{L}_{L1}$ is the pixel-wise L1 error and $\mathcal{L}_{D-SSIM}$ is the structural similarity loss [S5][S12, unofficial].
*   **Adaptive Density Control:** To manage complexity, Gaussians are dynamically added or removed [S1][S18, unofficial]:
    1.  **Clone:** Small Gaussians with large position gradients are cloned to fill under-reconstructed regions [S18, unofficial].
    2.  **Split:** Large Gaussians with large position gradients are split into two smaller Gaussians to capture fine details [S18, unofficial].
    3.  **Prune:** Gaussians with opacity below a threshold $\tau$ or excessively large screen-space size are removed [S18, unofficial].
    4.  **Opacity Reset:** Opacities are periodically reset to a small value to prune redundancies [S18, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_gaussian_splatting(gaussians, camera, image_shape):
    """
    Renders a scene defined by 3D Gaussians using tile-based rasterization.
    
    Args:
        gaussians: List of dicts, each containing:
            'mu': (3,) mean in world space
            'q': (4,) quaternion for rotation
            's': (3,) scale
            'alpha': scalar opacity
            'sh_coeffs': (3, (k+1)^2) spherical harmonic coefficients
        camera: Dict containing 'K' (3x3 intrinsic matrix) and 'W' (4x4 pose matrix)
        image_shape: Tuple (H, W)
        
    Returns:
        rendered_image: (H, W, 3) RGB image
    """
    H, W = image_shape
    tile_size = 16
    tiles_h = (H + tile_size - 1) // tile_size
    tiles_w = (W + tile_size - 1) // tile_size
    
    # Initialize output buffers
    color_buffer = np.zeros((H, W, 3))
    alpha_buffer = np.zeros((H, W))
    
    # Precompute camera matrices
    # W is world-to-camera. We assume W includes translation and rotation.
    # For projection, we need the Jacobian of the perspective projection.
    # Simplified: Assume standard pinhole model for Jacobian calculation
    K = camera['K']
    W_cam = camera['W']
    
    # Step 1: Project Gaussians to 2D
    projected_gaussians = []
    for g in gaussians:
        mu_3d = g['mu']
        
        # Transform mean to camera coordinates
        mu_4d = np.append(mu_3d, 1.0)
        mu_cam_4d = W_cam @ mu_4d
        mu_cam = mu_cam_4d[:3]
        
        # Check if behind camera
        if mu_cam[2] <= 0:
            continue
            
        # Compute 3D Covariance from q and s
        R = quat_to_rot_mat(g['q'])
        S = np.diag(g['s'])
        Sigma_3d = R @ (S @ S.T) @ R.T
        
        # Project Covariance to 2D
        # Jacobian J of perspective projection at mu_cam
        # Perspective projection: x' = K * [I | 0] * [R | t] * X
        # Linearization around mu_cam
        # J = d(proj)/d(X) approximated by first order Taylor
        # Standard derivation: J = K * [1/z 0 -x/z^2; 0 1/z -y/z^2; 0 0 0] * [R | t]
        # We only need the top 2x3 part for 2D covariance
        x, y, z = mu_cam
        J = np.array([
            [K[0,0]/z, 0, -K[0,0]*x/(z*z)],
            [0, K[1,1]/z, -K[1,1]*y/(z*z)]
        ])
        
        # Sigma_2d = J * W_rot * Sigma_3d * W_rot.T * J.T
        # W_rot is the upper 3x3 of W_cam (rotation part)
        W_rot = W_cam[:3, :3]
        Sigma_2d = J @ (W_rot @ Sigma_3d @ W_rot.T) @ J.T
        
        # Project mean to 2D
        mu_2d = K @ mu_cam_4d[:3] / mu_cam_4d[2]
        mu_2d = mu_2d[:2]
        
        projected_gaussians.append({
            'mu_2d': mu_2d,
            'Sigma_2d': Sigma_2d,
            'alpha': g['alpha'],
            'sh_coeffs': g['sh_coeffs'],
            'mu_cam_z': z
        })
        
    # Step 2: Tile-based assignment and sorting
    # Create a list of (tile_id, gaussian_index)
    tile_assignments = []
    
    for idx, pg in enumerate(projected_gaussians):
        mu_x, mu_y = pg['mu_2d']
        Sigma = pg['Sigma_2d']
        
        # Compute 3-sigma bounding box
        # Eigenvalues of Sigma give variance along principal axes
        eigenvalues = np.linalg.eigvalsh(Sigma)
        # 3-sigma extent
        extent_x = 3.0 * np.sqrt(eigenvalues[0])
        extent_y = 3.0 * np.sqrt(eigenvalues[1])
        
        min_x = int(np.floor(mu_x - extent_x))
        max_x = int(np.ceil(mu_x + extent_x))
        min_y = int(np.floor(mu_y - extent_y))
        max_y = int(np.ceil(mu_y + extent_y))
        
        # Clamp to image bounds
        min_x = max(0, min_x)
        max_x = min(W, max_x)
        min_y = max(0, min_y)
        max_y = min(H, max_y)
        
        for ty in range(min_y, max_y, tile_size):
            for tx in range(min_x, max_x, tile_size):
                tile_id = ty * tiles_w + (tx // tile_size)
                tile_assignments.append((tile_id, idx))
                
    # Sort by tile_id, then by depth (mu_cam_z)
    # We need to attach depth to the assignment
    depth_assignments = []
    for tile_id, idx in tile_assignments:
        depth_assignments.append((tile_id, projected_gaussians[idx]['mu_cam_z'], idx))
        
    depth_assignments.sort(key=lambda x: (x[0], x[1]))
    
    # Step 3: Render tiles
    # Group by tile
    current_tile = -1
    current_gaussians_in_tile = []
    
    for tile_id, depth, idx in depth_assignments:
        if tile_id != current_tile:
            # Render previous tile if exists
            if current_tile != -1:
                render_tile(current_tile, current_gaussians_in_tile, projected_gaussians, color_buffer, alpha_buffer, image_shape)
            current_tile = tile_id
            current_gaussians_in_tile = []
        current_gaussians_in_tile.append(idx)
        
    # Render last tile
    if current_tile != -1:
        render_tile(current_tile, current_gaussians_in_tile, projected_gaussians, color_buffer, alpha_buffer, image_shape)
        
    return color_buffer

def render_tile(tile_id, gaussian_indices, projected_gaussians, color_buffer, alpha_buffer, image_shape):
    H, W = image_shape
    tile_size = 16
    ty = tile_id // ((W + tile_size - 1) // tile_size)
    tx = tile_id % ((W + tile_size - 1) // tile_size)
    
    y_start = ty * tile_size
    x_start = tx * tile_size
    
    # Get tile bounds
    y_end = min(H, y_start + tile_size)
    x_end = min(W, x_start + tile_size)
    
    # For each pixel in the tile
    for y in range(y_start, y_end):
        for x in range(x_start, x_end):
            T = 1.0
            C = np.array([0.0, 0.0, 0.0])
            
            for idx in gaussian_indices:
                pg = projected_gaussians[idx]
                mu_x, mu_y = pg['mu_2d']
                Sigma = pg['Sigma_2d']
                alpha = pg['alpha']
                sh_coeffs = pg['sh_coeffs']
                
                # Compute distance vector
                dx = x - mu_x
                dy = y - mu_y
                
                # Compute Gaussian value
                # power = -0.5 * (x, y) . Sigma^{-1} . (x, y)
                # Use conic matrix for efficiency: C = Sigma^{-1}
                try:
                    Sigma_inv = np.linalg.inv(Sigma)
                except np.linalg.LinAlgError:
                    continue
                    
                power = -0.5 * (dx * Sigma_inv[0,0] * dx + 2 * dx * Sigma_inv[0,1] * dy + dy * Sigma_inv[1,1] * dy)
                
                if power > 0:
                    continue # Gaussian value is negligible
                    
                g_val = np.exp(power)
                alpha_eff = alpha * g_val
                
                if alpha_eff < 1e-4:
                    continue
                
                # Compute color from SH
                # View direction is needed. For simplicity, assume view direction is camera forward or normalized vector from center to pixel
                # In practice, SH evaluation requires the viewing direction vector v
                # v = normalize(camera_center - pixel_world_pos)
                # Here we approximate v as the camera's forward direction or use a placeholder
                # For a complete implementation, pixel world position must be back-projected
                v = np.array([0.0, 0.0, 1.0]) # Placeholder for view direction
                color = evaluate_sh(sh_coeffs, v)
                
                # Alpha blending
                C += color * alpha_eff * T
                T *= (1.0 - alpha_eff)
                
                if T < 1e-4:
                    break
            
            color_buffer[y, x] = C
            alpha_buffer[y, x] = 1.0 - T

def evaluate_sh(sh_coeffs, v):
    """Evaluate spherical harmonics for a given view direction v."""
    # Simplified band 0 evaluation
    # c = 0.5 + C0 * f_dc
    C0 = 0.28209479177387814
    f_dc = sh_coeffs[:, 0] # First 3 coefficients are DC
    color = np.clip(0.5 + C0 * f_dc, 0.0, 1.0)
    return color

def quat_to_rot_mat(q):
    """Convert quaternion to rotation matrix."""
    w, x, y, z = q
    return np.array([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*w*z, 2*x*z + 2*w*y],
        [2*x*y + 2*w*z, 1 - 2*x*x - 2*z*z, 2*y*z - 2*w*x],
        [2*x*z - 2*w*y, 2*y*z + 2*w*x, 1 - 2*x*x - 2*y*y]
    ])
```

## Subtopics

### [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md)
Defines the primitive shape where amplitude equals opacity.

### [Row-constant hoisting for Gaussian Splatting Rasterization](1195-row-constant-hoisting-for-gaussian-splatting-rasterization.md)
Optimization for the inner pixel loop.

### [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md)
Projection method with epsilon stabilization.

### [Gaussian in the Wild (GS-W)](1386-gaussian-in-the-wild-gs-w.md)
Method for unconstrained image collections separating material and environment.

### [3DGUT](1474-3dgut.md)
Uses unscented transform for projection.

### [Transformer-based Reconstruction](1794-transformer-based-reconstruction.md)
Feed-forward network replacing iterative optimization.

### [3D Gaussian Ellipsoid Rendering](1869-3d-gaussian-ellipsoid-rendering.md)
Ray-traced volume rendering of Gaussians.

### [Stochastic Gaussian Splatting](6599-stochastic-gaussian-splatting.md)
Probabilistic sampling variants.

### [Rendering Implementation](7155-rendering-implementation.md)
Complete pipeline for projection and compositing.

### [Contribution-based ordering](8748-contribution-based-ordering.md)
Post-training sorting for progressive rendering.

### [3D Convex Splatting](8758-3d-convex-splatting.md)
Replaces Gaussians with convex shapes.

### [Feed-forward 3D Gaussian Splatting](9932-feed-forward-3d-gaussian-splatting.md)
Predicts Gaussians from sparse images via Transformer.

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N \cdot T)$ where $N$ is the number of Gaussians and $T$ is the number of tiles, but effectively linear in the number of visible primitives per pixel due to tile-based culling and sorting [S5, S15]. Training is iterative and depends on the number of Gaussians and images [S1].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of Gaussians. Each Gaussian stores position, covariance (via q, s), opacity, and SH coefficients [S5][S7, unofficial].
*   **Performance:** Enables real-time rendering (e.g., 100+ FPS) on modern GPUs [S1, S5].
*   **Strengths:** High visual quality, real-time rendering, explicit representation allows for easy editing and compression [S4, S5].
*   **Limitations:** Large storage requirements compared to NeRFs for high-fidelity scenes [S4]. Approximations in projection and sorting can cause artifacts (e.g., popping) [S2].
*   **When to use:** Real-time novel view synthesis, interactive applications, scenes with complex geometry and appearance [S1, S4].
*   **When not to use:** Memory-constrained environments where implicit representations are preferred, or when exact volumetric accuracy is required over speed [S2, S4].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1, S2]. It builds on earlier work in volume splatting, such as Elliptical Weighted Average (EWA) splatting [S2][S3, unofficial], and point-based rendering methods [S2]. The method was developed to address the computational inefficiencies of NeRFs, which rely on dense ray sampling and neural network queries [S1][S18, unofficial]. By using explicit 3D Gaussians and differentiable rasterization, 3DGS achieves a balance between the quality of neural rendering and the speed of traditional graphics pipelines [S5].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S3] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S4] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S5] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S6] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S7] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S8] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27 (unofficial)
[S9] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
[S10] Refining Gaussian Splatting: A Volumetric Densification Approach, https://doi.org/10.24132/CSRN.2025-6, fetched 2026-10-26
[S11] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S12] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
[S13] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
[S14] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S15] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S16] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S17] UltraGauss: Ultrafast Gaussian Reconstruction of 3D Ultrasound Volumes, https://doi.org/10.48550/arXiv.2505.05643, fetched 2026-10-27 (unofficial)
[S18] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S19] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S20] Neural Rendering: Graphics Learned From Images, https://www.ultralytics.com/glossary/neural-rendering, fetched 2026-10-27
[S21] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S22] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S23] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S24] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S25] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S26] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S27] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
[S28] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
[S29] Differentiable Rendering - 可微渲染：从2D 学习3D, https://www.ultralytics.com/zh/glossary/differentiable-rendering, fetched 2026-10-30
[S30] 从体渲染到3D Gaussian Splatting：理论与应用, https://www.alphaxiv.org/zh/abs/2510.18101, fetched 2026-10-30 (unofficial)
[S31] Gaussian Splatting - 高斯泼溅：实时3D 渲染, https://www.ultralytics.com/zh/glossary/gaussian-splatting, fetched 2026-10-30
[S32] 众趣分享丨浅谈空间计算技术基础---3D 高斯泼溅, https://www.3dnest.cn/article/1280.html, fetched 2026-10-30 (unofficial)
