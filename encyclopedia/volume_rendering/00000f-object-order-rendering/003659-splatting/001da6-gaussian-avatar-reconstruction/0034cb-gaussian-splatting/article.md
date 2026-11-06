# Gaussian Splatting

Also called: 3D Gaussian Splatting (3DGS), Vanilla 3DGS rendering

## Overview
Gaussian Splatting is a novel-view synthesis method that represents a scene as an explicit collection of anisotropic 3D Gaussians carrying opacity and view-dependent color, optimized directly from calibrated images [S5, unofficial]. It solves the trade-off between the high fidelity of neural radiance fields (NeRF) and the real-time performance of point-based rendering by replacing implicit neural queries and ray marching with a tile-based differentiable rasterizer [S5, unofficial]. Unlike NeRF, which requires hundreds of MLP evaluations per pixel and supervision of empty space, 3DGS uses explicit primitives initialized on geometry, avoiding costly queries in empty space and enabling real-time rasterization [S1].

## How it works
**Inputs:** A set of posed input images $\{I_i\}$ and their corresponding camera poses.
**Outputs:** A rendered image $I_{\theta}$, depth map, and mask for a given camera view; or the optimized set of Gaussian parameters $\theta$ for novel view synthesis.

**Data Structures:**
The scene is represented by a set of $M$ 3D Gaussians. Each Gaussian $g_i$ is defined by the following parameters [S1][S12, unofficial]:
*   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Center position in world space.
*   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix describing shape and orientation. To ensure positive semi-definiteness during optimization, $\boldsymbol{\Sigma}_i$ is parameterized via factorization [S5, unofficial]:
    $$ \boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top $$
    where $\mathbf{R}_i$ is a rotation matrix derived from a normalized quaternion $\mathbf{q}_i \in \mathbb{R}^4$ ($q_r^2 + q_i^2 + q_j^2 + q_k^2 = 1$), and $\mathbf{S}_i$ is a diagonal scaling matrix constructed from a scale vector $\mathbf{s}_i \in \mathbb{R}^3$ [S5, S12, unofficial]. The scale vector is typically optimized in log space [S4, unofficial].
*   $\sigma_i \in \mathbb{R}$: Opacity (logit).
*   $\mathbf{h}_i \in \mathbb{R}^{4 \times 3}$: Spherical Harmonic (SH) coefficients for view-dependent color. Band 3 (48 coefficients per splat for RGB) is standard [S4, unofficial].

**Rendering Pipeline:**
The rendering process projects 3D Gaussians to 2D screen-space ellipses and composites them via alpha blending [S2][S3, unofficial].

1.  **Initialization:** Initialize Gaussian centers $\boldsymbol{\mu}_i$ and colors from a sparse point cloud obtained via Structure-from-Motion (SfM) [S1][S5, unofficial]. Random initialization is avoided as it introduces artifacts and floaters [S5, unofficial].
2.  **3D-to-2D Projection:** Project each 3D Gaussian into the 2D image plane using an affine approximation of the perspective projection [S8][S12, unofficial].
    *   Transform the center to camera space: $\boldsymbol{\mu}_i^c = \mathbf{W} \boldsymbol{\mu}_i$, where $\mathbf{W}$ is the camera pose matrix [S12, unofficial].
    *   Compute the Jacobian $\mathbf{J} \in \mathbb{R}^{2 \times 3}$ of the perspective projection $\phi_{\mathbf{K}}$ at $\boldsymbol{\mu}_i^c$ [S12, unofficial].
    *   Compute the 2D mean: $\tilde{\boldsymbol{\mu}}_i = \phi_{\mathbf{K}}(\boldsymbol{\mu}_i^c)$ [S1].
    *   Compute the 2D covariance: $\tilde{\boldsymbol{\Sigma}}_i = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma}_i \mathbf{W}^\top \mathbf{J}^\top$ [S1][S12, unofficial].
3.  **Tile-Based Sorting:**
    *   Partition the image into tiles (e.g., $16 \times 16$ pixels) [S8].
    *   Compute the 3-sigma bounding box for each Gaussian in screen space. Assign Gaussians intersecting the view frustum to overlapping tiles [S8][S11, unofficial].
    *   Sort the Gaussians within each tile by depth (z-value) relative to the camera [S8][S11, unofficial].
4.  **Alpha Compositing (Rasterization):**
    *   For each pixel, traverse the sorted list of overlapping Gaussians in back-to-front order [S7, unofficial].
    *   Evaluate the 2D Gaussian at the pixel coordinate $\mathbf{x}$:
        $$ \tilde{\mathcal{G}}_i(\mathbf{x}) = \exp\left(-\frac{1}{2} (\mathbf{x} - \tilde{\boldsymbol{\mu}}_i)^\top \tilde{\boldsymbol{\Sigma}}_i^{-1} (\mathbf{x} - \tilde{\boldsymbol{\mu}}_i)\right) $$
    *   Compute the effective alpha for the $i$-th Gaussian: $\alpha_i = \sigma_i \tilde{\mathcal{G}}_i(\mathbf{x})$ [S1][S12, unofficial]. Clamp $\alpha_i$ to a maximum of 0.99 to prevent full opacity from blocking subsequent splats entirely [S11, unofficial].
    *   Compute the color $c_i$ by evaluating the SH coefficients with the viewing direction $\mathbf{d}$ from the camera to the Gaussian center [S7, unofficial]. For band 0: $c_i = \text{clamp}(0.5 + C_0 \cdot f_{dc}, 0, 1)$, where $C_0 \approx 0.282$ [S4, unofficial].
    *   Accumulate color using alpha blending:
        $$ C_{\mathbf{x}} = \sum_{i=1}^{N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance [S9][S12, unofficial].
    *   Apply a background color $c_{BG}$ (default white) weighted by the remaining transmittance [S9].

**Optimization:**
*   **Loss:** Minimize photometric loss between rendered image $I_{\theta,i}$ and input view $I_i$:
    $$ \mathcal{L}(\theta) = \frac{1}{N} \sum_{i=1}^N \left( \| \mathbf{I}_{\theta,i} - \mathbf{I}_i \|_1 + \lambda_{SSIM} \cdot SSIM(\mathbf{I}_{\theta,i}, \mathbf{I}_i) \right) $$
    [S6, S12, unofficial].
*   **Adaptive Density Control:** During training, dynamically adjust the number of Gaussians [S5, unofficial]:
    *   **Split:** Split Gaussians that are too large (based on gradient magnitude) to capture high-frequency details.
    *   **Clone:** Clone Gaussians that underfit local detail.
    *   **Prune:** Remove Gaussians with persistently low opacity [S1][S12, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_gaussians(gaussians, camera, image_width, image_height, tile_size=16):
    """
    Renders a scene of 3D Gaussians using tile-based rasterization.
    
    Args:
        gaussians: List of dicts with keys 'mu' (3,), 'scale' (3,), 'rot' (4,), 
                   'opacity' (float), 'sh_coeffs' (48,).
        camera: Dict with 'W' (4x4 pose matrix), 'K' (3x3 intrinsics).
        image_width, image_height: int.
        tile_size: int.
        
    Returns:
        image: (H, W, 3) uint8 array.
    """
    # 1. Precompute 2D Gaussians
    projected_gaussians = []
    for g in gaussians:
        # Transform center to camera space
        mu_world = g['mu']
        mu_cam = camera['W'] @ np.append(mu_world, 1)
        mu_cam = mu_cam[:3] / mu_cam[2]
        
        # Compute 3D Covariance from scale and rotation
        R = quat_to_rot_mat(g['rot'])
        S = np.diag(np.exp(g['scale'])) # Scale is stored in log space
        Sigma_3d = R @ (S @ S.T) @ R.T
        
        # Project Covariance to 2D
        # Jacobian of perspective projection at mu_cam
        # K is 3x3, W is 4x4. 
        # J = d(K * [x,y,z,1]^T / z) / d[x,y,z]
        # Simplified: J = K[:2, :3] * (1/z - [x,y,z]/z^2 * z_deriv) ... 
        # Standard derivation: J = K[:2, :3] @ (I - [mu_cam, 1] * [0,0,1]^T / mu_cam[2])
        # Or more simply using the chain rule on the projection function.
        # Let's use the standard approximation from [S12, unofficial]:
        # J = d(phi_K(W * mu)) / d(mu)
        # phi_K(p) = K * [p.x/p.z, p.y/p.z, 1]
        # d(phi_K)/d(p) = K[:2, :3] * (1/p.z - p[:3]/p.z^2 * [0,0,1])
        # Let T = W. mu_cam = T @ mu_world.
        # d(mu_cam)/d(mu_world) = T[:3, :3]
        
        # Compute Jacobian J_proj (2x3) for perspective projection at mu_cam
        z = mu_cam[2]
        if z <= 0: continue # Behind camera
        
        # Derivative of perspective divide
        # d(x/z)/dx = 1/z, d(x/z)/dz = -x/z^2
        # d(y/z)/dy = 1/z, d(y/z)/dz = -y/z^2
        J_div = np.array([
            [1/z, 0, -mu_cam[0]/z**2],
            [0, 1/z, -mu_cam[1]/z**2]
        ])
        
        # Full Jacobian J = K[:2, :3] @ J_div @ T[:3, :3]
        J = camera['K'][:2, :3] @ J_div @ camera['W'][:3, :3]
        
        # 2D Covariance: Sigma_2d = J @ Sigma_3d @ J.T
        Sigma_2d = J @ Sigma_3d @ J.T
        
        # 2D Mean
        mu_2d = camera['K'][:2, :3] @ np.append(mu_cam, 1)
        mu_2d = mu_2d / mu_2d[2] # Perspective division
        
        # Compute bounding box (3-sigma)
        eigenvalues, eigenvectors = np.linalg.eigh(Sigma_2d)
        # 3-sigma extent
        extents = 3.0 * np.sqrt(eigenvalues)
        min_x = mu_2d[0] - extents[0]
        max_x = mu_2d[0] + extents[0]
        min_y = mu_2d[1] - extents[1]
        max_y = mu_2d[1] + extents[1]
        
        # Clip to image bounds
        min_x = max(0, min_x)
        max_x = min(image_width, max_x)
        min_y = max(0, min_y)
        max_y = min(image_height, max_y)
        
        if min_x >= max_x or min_y >= max_y:
            continue
            
        projected_gaussians.append({
            'mu_2d': mu_2d,
            'Sigma_2d': Sigma_2d,
            'opacity': g['opacity'],
            'sh_coeffs': g['sh_coeffs'],
            'z': mu_cam[2],
            'min_x': min_x, 'max_x': max_x,
            'min_y': min_y, 'max_y': max_y
        })
        
    # 2. Tile-based Sorting
    # Create tiles
    num_tiles_x = (image_width + tile_size - 1) // tile_size
    num_tiles_y = (image_height + tile_size - 1) // tile_size
    tiles = [[[] for _ in range(num_tiles_x)] for _ in range(num_tiles_y)]
    
    for g in projected_gaussians:
        # Determine which tiles this gaussian overlaps
        tile_min_x = int(g['min_x'] // tile_size)
        tile_max_x = int(g['max_x'] // tile_size)
        tile_min_y = int(g['min_y'] // tile_size)
        tile_max_y = int(g['max_y'] // tile_size)
        
        for ty in range(tile_min_y, tile_max_y + 1):
            for tx in range(tile_min_x, tile_max_x + 1):
                if 0 <= ty < num_tiles_y and 0 <= tx < num_tiles_x:
                    tiles[ty][tx].append(g)
                    
    # Sort each tile's gaussians by depth (back to front)
    for ty in range(num_tiles_y):
        for tx in range(num_tiles_x):
            tiles[ty][tx].sort(key=lambda g: g['z'], reverse=True)
            
    # 3. Rasterization
    image = np.zeros((image_height, image_width, 3), dtype=np.float32)
    transmittance = np.ones((image_height, image_width), dtype=np.float32)
    
    bg_color = np.array([1.0, 1.0, 1.0])
    
    for ty in range(num_tiles_y):
        for tx in range(num_tiles_x):
            tile_gaussians = tiles[ty][tx]
            
            # Pixel coordinates in this tile
            y_start = ty * tile_size
            x_start = tx * tile_size
            
            for y in range(y_start, min(y_start + tile_size, image_height)):
                for x in range(x_start, min(x_start + tile_size, image_width)):
                    if transmittance[y, x] < 1e-4:
                        continue
                        
                    color_acc = np.array([0.0, 0.0, 0.0])
                    
                    for g in tile_gaussians:
                        # Check if pixel is within bounding box
                        if not (g['min_x'] <= x < g['max_x'] and g['min_y'] <= y < g['max_y']):
                            continue
                            
                        # Evaluate 2D Gaussian
                        dx = x - g['mu_2d'][0]
                        dy = y - g['mu_2d'][1]
                        # Power = -0.5 * (x^T Sigma^-1 x)
                        # Using conic matrix C = Sigma^-1
                        try:
                            C = np.linalg.inv(g['Sigma_2d'])
                        except:
                            continue
                            
                        power = -0.5 * (C[0,0]*dx**2 + C[1,1]*dy**2 + 2*C[0,1]*dx*dy)
                        
                        if power > 0.0:
                            continue # Outside effective area (exp(negative) < 1, but if power > 0, exp(power) > 1 which is invalid for density)
                            
                        g_val = np.exp(power)
                        alpha = g['opacity'] * g_val
                        alpha = min(alpha, 0.99) # Clamp alpha
                        
                        # Compute color from SH
                        # View direction from camera to gaussian center
                        # Approximate view direction as camera to pixel or gaussian center
                        # For simplicity, use gaussian center direction
                        cam_pos = np.linalg.inv(camera['W'])[:3, 3]
                        view_dir = g['mu_2d'] # This is screen space, need world space view dir
                        # Reconstruct world pos from screen pos? 
                        # Standard 3DGS evaluates SH at the viewing direction from camera to Gaussian center
                        # We have mu_cam. View dir is -mu_cam (normalized)
                        v_dir = -mu_cam / np.linalg.norm(mu_cam)
                        color = eval_sh(g['sh_coeffs'], v_dir)
                        
                        # Blend
                        color_acc = color_acc + (1 - transmittance[y, x]) * alpha * color
                        transmittance[y, x] = transmittance[y, x] * (1 - alpha)
                        
                        if transmittance[y, x] < 1e-4:
                            break
                            
                    # Add background
                    final_color = color_acc + transmittance[y, x] * bg_color
                    image[y, x] = final_color
                    
    return np.clip(image, 0, 1).astype(np.uint8)

def quat_to_rot_mat(q):
    # q = [w, x, y, z]
    w, x, y, z = q
    return np.array([
        [1 - 2*(y**2 + z**2), 2*(x*y - w*z), 2*(x*z + w*y)],
        [2*(x*y + w*z), 1 - 2*(x**2 + z**2), 2*(y*z - w*x)],
        [2*(x*z - w*y), 2*(y*z + w*x), 1 - 2*(x**2 + y**2)]
    ])

def eval_sh(coeffs, v_dir):
    # Simplified Band 0 evaluation
    # C0 = 0.28209479177387814
    C0 = 0.28209479177387814
    # coeffs shape (48,) -> reshape to (4, 3)
    coeffs = coeffs.reshape(4, 3)
    # Band 0 is coeffs[0]
    dc = coeffs[0]
    color = 0.5 + C0 * dc
    return np.clip(color, 0, 1)
```

## Subtopics

### [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md)
The Unnormalized 3D Gaussian Function is a Gaussian representation used in 3D Gaussian Splatting where the amplitude $a$ at the mean $\boldsymbol{\mu}$ equals the opacity, representing the peak capacity to block or emit light. It differs from the [Normalized 3D Gaussian Function](1064-normalized-3d-gaussian-function.md) by parameterizing the function via its peak value rather than its integral, making $a$ the intensity at the center rather than the total light capacity.

### [Row-constant hoisting for Gaussian Splatting Rasterization](1195-row-constant-hoisting-for-gaussian-splatting-rasterization.md)
Row-constant hoisting is an optimization for the inner pixel loop of a Gaussian splatting rasterizer that hoists row-constant terms outside the inner (column) loop to minimize arithmetic operations per pixel. It reduces the computational cost of evaluating the 2D Gaussian exponent by precomputing terms dependent only on the row coordinate.

### [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md)
This method projects a 3D Gaussian primitive, defined by a mean and a covariance matrix in world space, onto the 2D image plane to create a screen-space Gaussian splat. It solves the problem of rendering millions of 3D Gaussians efficiently by avoiding ray marching; instead, it approximates the non-linear perspective projection with a local affine transformation, preserving the Gaussian form as a 2D ellipse. It sets itself apart from naive point-splatting by accounting for anisotropic spread and orientation, and from exact perspective projection by using a first-order Taylor expansion (Jacobian) to maintain computational tractability and differentiability.

### [Gaussian in the Wild (GS-W)](1386-gaussian-in-the-wild-gs-w.md)
Gaussian in the Wild (GS-W) is a novel view synthesis method for unconstrained image collections that explicitly separates scene appearance into intrinsic material properties and dynamic environmental factors (such as illumination and weather) per 3D Gaussian point. It differs from standard 3D Gaussian Splatting and NeRF-based methods by avoiding global latent embeddings, which allows it to capture local high-frequency changes without blurring intrinsic textures with dynamic variations. The method employs adaptive sampling from image features and 2D visibility maps to mitigate transient occlusions and photometric variations.

### [3DGUT](1474-3dgut.md)
3DGUT is a Gaussian projection technique that employs an unscented transform to improve the affine approximation of the projection operator, providing more precise Gaussian projection onto the image plane. It addresses the accuracy limitations of the standard linearization used in Gaussian Splatting by capturing higher-order statistical moments of the projection, thereby reducing approximation errors in the resulting 2D covariance.

### [Transformer-based Reconstruction](1794-transformer-based-reconstruction.md)
Transformer-based Reconstruction refers to a family of feed-forward neural network methods that directly predict 3D Gaussian Splatting (3DGS) parameters from 2D image inputs, bypassing the iterative optimization required by standard 3DGS [G]. Unlike iterative approaches that refine a scene representation over time, these models infer the scene geometry and appearance in a single forward pass [G]. The family is distinguished from sibling rasterization and rendering optimizations by its focus on the generative reconstruction pipeline rather than the rendering speed or mathematical stability of the Gaussian primitives themselves [G].

### [3D Gaussian Ellipsoid Rendering](1869-3d-gaussian-ellipsoid-rendering.md)
3D Gaussian Ellipsoid Rendering (VoGE) is a differentiable volume rendering method that represents scenes as a sum of 3D Gaussian ellipsoids and computes image synthesis via ray tracing rather than rasterization. It solves the occlusion ambiguity inherent in distance-based blending by accumulating volume density distributions along viewing rays, enabling realistic interaction between overlapping objects. Unlike its sibling [Gaussian Splatting](946-gaussian-splatting.md), which approximates volume rendering through 2D rasterization of projected splats, VoGE replaces rasterization with ray tracing to capture exact volume density distributions for blending.

### [Stochastic Gaussian Splatting](6599-stochastic-gaussian-splatting.md)
Stochastic Gaussian Splatting is a rendering method for massive 3D Gaussian Splatting (3DGS) scenes that eliminates the computational bottlenecks of sorting and tile-based rasterization by using stochastic sampling. It samples opaque points from Gaussians and splats them using atomic operations, applying a derived opacity correction to ensure unbiased convergence to the true volume rendering result. Unlike deterministic approaches or culling-based methods like SplatShop, it renders all Gaussians within the frustum, enabling real-time display of hundreds of millions of Gaussians on consumer hardware.

### [Rendering Implementation](7155-rendering-implementation.md)
Rendering Implementation defines the computational pipeline that transforms a learned scene representation of $M$ 3D Gaussians into a 2D image. It solves the problem of efficiently evaluating the volume rendering equation by projecting 3D Gaussians to 2D screen-space ellipses and compositing them via alpha blending. It differs from its siblings by providing the core differentiable kernel that bridges the 3D scene representation and the 2D image plane, whereas methods like [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md) focus on training efficiency and [Hybrid Transparency](3294-hybrid-transparency.md) on specific blending artifacts.

### Gsplat Transformation
To transform a Gsplat: 1. Copy input `gsplat` to output. 2. If `isGsplatActive(gsplat.flags)` is true: a. If `scale` is provided, multiply `gsplat.center` and `gsplat.scales` by `scale`. b. If `rotate` (vec4 quaternion) is provided, rotate `gsplat.center` using `quatVec(rotate, gsplat.center)` and update `gsplat.quaternion` using `quatQuat(rotate, gsplat.quaternion)`. c. If `translate` (vec3) is provided, add `translate` to `gsplat.center`. d. If `recolor` (vec4) is provided, multiply `gsplat.rgba` by `recolor` [S15, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The scene requires storing $M$ Gaussians. Complex scenes require approximately 200,000–500,000 Gaussians [S12, unofficial]. Each Gaussian stores position (12 bytes), scale (12 bytes), rotation (16 bytes), opacity (4 bytes), and SH coefficients (48 coefficients * 4 bytes * 3 channels = 576 bytes for Band 3). Total memory is roughly $M \times 620$ bytes. Default parameter settings may use approximately 6GB of GPU memory [S9].
*   **Time:** Rendering is $O(M)$ per pixel in the worst case, but tile-based culling reduces this significantly. Sorting is $O(M \log M)$ or $O(M)$ with radix sort [S8]. Training time is minutes, compared to hours/days for NeRF [S5, unofficial].

**Measured Performance:**
*   On the Mip-NeRF360 benchmark, the method reaches 27.21 dB PSNR and 0.214 LPIPS at 134 fps [S5, unofficial].
*   Rendering performance target is $\ge 30$ fps at 1080p resolution on consumer hardware [S5, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering, high fidelity, explicit representation allows for easy editing and compression [S5, unofficial].
*   **Limitations:** Standard 3DGS uses a dilation-based filter which fails to suppress high-frequency artifacts when varying focal length or camera distance [S12, unofficial]. It employs an emission-absorption model instead of a scattering model, which limits performance on highly reflective surfaces and prevents relighting [S12, unofficial]. Explicit per-pixel depth sorting incurs substantial computational overhead and causes temporal artifacts such as flickering and popping when Gaussian order changes [S14, unofficial]. Self-attenuation is not treated correctly [S2].

**When to Use:**
*   Use for novel-view synthesis, virtual walkthroughs, free-viewpoint replays, digital twins, and content capture for VR and AR [S5, unofficial].
*   Do not use when accurate volumetric rendering (e.g., for medical imaging or highly reflective surfaces) is required, or when relighting is needed [S2][S12, unofficial].

## Background
Gaussian Splatting was introduced by Kerbl et al. in 2023 [S5, unofficial]. It builds on EWA Volume Splatting [S2] and several works on point-based rendering [S5, unofficial]. It was developed to address the computational costs of NeRF and the lack of fidelity in previous point-based splatting methods [S5, unofficial]. It is used for novel-view synthesis, virtual walkthroughs, free-viewpoint replays, digital twins, and content capture for VR and AR [S5, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S3] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S4] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S5] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S6] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
[S7] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
[S8] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S9] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S10] UltraGauss: Ultrafast Gaussian Reconstruction of 3D Ultrasound Volumes, https://doi.org/10.48550/arXiv.2505.05643, fetched 2026-10-27 (unofficial)
[S11] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S12] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S13] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S14] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
[S15] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
