# Gaussian Splatting

Also called: 3D Gaussian Splatting (3DGS), 3D GS

## Overview
Gaussian Splatting is a method for 3D scene representation and novel view synthesis that models a scene as a collection of anisotropic 3D Gaussians, each defined by position, covariance, opacity, and view-dependent color [S1][S3, unofficial][S5][S12]. It solves the computational bottleneck of implicit neural radiance fields (NeRFs) by replacing ray marching with a differentiable, tile-based rasterization pipeline [S1][S19][S21]. Unlike NeRFs, which query dense volumes along rays, Gaussian Splatting projects 3D ellipsoids to 2D screen-space footprints (splats) and composites them via alpha blending, enabling real-time rendering speeds [S1, S7][S16, unofficial]. It sets itself apart from its siblings by using explicit, learnable Gaussian primitives optimized directly from images, rather than implicit neural networks or fixed geometric meshes [S6][S12].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed input images $\{I_i\}$ and a sparse Structure-from-Motion (SfM) point cloud providing initial camera poses and 3D points [S1][S15, unofficial][S34].
*   **Outputs:** A set of $N$ 3D Gaussian primitives $\mathcal{G} = \{g_i\}_{i=1}^N$, where each $g_i$ is parameterized by a mean $\boldsymbol{\mu}_i \in \mathbb{R}^3$, a covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$, an opacity $\alpha_i \in [0, 1]$, and spherical harmonic (SH) coefficients $\mathbf{c}_i$ for color [S1][S3, unofficial][S6][S23, unofficial]. The method also outputs rendered novel views [S1].

**Data Structures:**
Each Gaussian $g_i$ stores:
*   $\boldsymbol{\mu}_i$: 3D position (mean).
*   $\mathbf{q}_i$: A quaternion (4 floats) representing rotation, from which the rotation matrix $\mathbf{R}_i$ is derived [S6][S25].
*   $\mathbf{s}_i$: A 3D vector representing scale, from which the scaling matrix $\mathbf{S}_i = \text{diag}(\mathbf{s}_i)$ is derived [S6][S25].
*   $\alpha_i$: Opacity (1 float).
*   $\mathbf{c}_{i, l}$: SH coefficients for band $l$ (view-dependent color). For band 3, this is 48 floats (3 RGB channels $\times$ 16 coefficients) [S8, S12].
*   The covariance matrix is reconstructed as $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$ [S6][S25]. This parameterization ensures $\boldsymbol{\Sigma}_i$ remains positive semi-definite during optimization [S6][S25].

**Rendering Pipeline:**
The rendering process consists of four main stages: Initialization, Projection, Sorting, and Compositing [S1, S6][S16, unofficial].

1.  **Initialization:**
    Gaussians are initialized at the 3D points of the SfM cloud [S1]. The mean $\boldsymbol{\mu}_i$ is set to the point position [S1]. The initial scale $\mathbf{s}_i$, rotation $\mathbf{q}_i$, and opacity $\alpha_i$ are set to small constants or identity, as standard practice [G]. Color $\mathbf{c}_i$ is initialized by averaging or median-filtering pixel colors from the input images where the point is visible [S36, unofficial].

2.  **Projection (Splatting):**
    Each 3D Gaussian is projected to the 2D image plane [S32].
    *   **Coordinate Transformation:** The 3D mean $\boldsymbol{\mu}_i$ is transformed to camera coordinates: $\boldsymbol{\mu}_i^c = \mathbf{W} \boldsymbol{\mu}_i$, where $\mathbf{W}$ is the world-to-camera transformation matrix [S6][S22, unofficial].
    *   **2D Mean:** The projected 2D mean is computed via perspective projection $\phi_{\mathbf{K}}$ (using camera intrinsics $\mathbf{K}$): $\tilde{\boldsymbol{\mu}}_i = \phi_{\mathbf{K}}(\boldsymbol{\mu}_i^c)$ [S1][S22, unofficial].
    *   **2D Covariance:** The perspective projection is non-affine [S31, unofficial]. It is approximated by a local affine transformation using the Jacobian $\mathbf{J} \in \mathbb{R}^{2 \times 3}$ of the projection at $\boldsymbol{\mu}_i^c$ [S1, S6]. The 2D covariance matrix $\tilde{\boldsymbol{\Sigma}}_i$ is computed as:
        $$ \tilde{\boldsymbol{\Sigma}}_i = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma}_i \mathbf{W}^T \mathbf{J}^T $$
        Note: $\mathbf{W}$ in the covariance formula refers to the rotational component of the world-to-camera transform if $\boldsymbol{\Sigma}_i$ is defined in world space, or the full linearization if defined in camera space [S6].
    *   **Conic Matrix:** To avoid matrix inversion in the fragment shader, the inverse 2D covariance is computed as the conic matrix $\mathbf{C}_i = \tilde{\boldsymbol{\Sigma}}_i^{-1}$ [S20].

3.  **Sorting:**
    *   **Frustum Culling:** Gaussians outside the camera frustum are discarded [S6].
    *   **Tile Assignment:** The image is divided into non-overlapping tiles (typically $16 \times 16$ pixels) [S6][S16, unofficial]. For each Gaussian, an axis-aligned bounding box (AABB) is computed in screen space [S6]. The Gaussian is assigned to all tiles it intersects [S6][S30, unofficial].
    *   **Depth Sorting:** Within each tile, the assigned Gaussians are sorted by depth (distance from camera center to $\boldsymbol{\mu}_i$) in back-to-front order [S6][S16, S30, unofficial]. This is typically done using a GPU-optimized radix sort on a packed key of (tile ID, depth) [S6].

4.  **Compositing (Alpha Blending):**
    For each pixel in a tile, the sorted list of overlapping Gaussians is traversed front-to-back (or back-to-front depending on implementation, but standard is front-to-back accumulation of transmittance) [S1, S6].
    *   **2D Gaussian Evaluation:** For a pixel at screen coordinate $\mathbf{x}$, the value of the projected 2D Gaussian is:
        $$ \mathcal{G}_i^{2D}(\mathbf{x}) = \exp\left(-\frac{1}{2} (\mathbf{x} - \tilde{\boldsymbol{\mu}}_i)^T \tilde{\boldsymbol{\Sigma}}_i^{-1} (\mathbf{x} - \tilde{\boldsymbol{\mu}}_i)\right) $$
        In practice, this is computed using the conic matrix $\mathbf{C}_i = [C_{00}, C_{01}, C_{11}]$ and local coordinates $(x, y) = \mathbf{x} - \tilde{\boldsymbol{\mu}}_i$:
        $$ \text{power} = -0.5 (C_{00} x^2 + C_{11} y^2) - C_{01} xy $$
        $$ \mathcal{G}_i^{2D}(\mathbf{x}) = \exp(\text{power}) $$
        [S20][S30, unofficial].
    *   **Effective Opacity:** The effective opacity $\alpha'_i$ contributed by the Gaussian at the pixel is the product of the learned opacity $\alpha_i$ and the 2D Gaussian value:
        $$ \alpha'_i = \alpha_i \mathcal{G}_i^{2D}(\mathbf{x}) $$
        [S6][S19][S30, unofficial].
    *   **Color Retrieval:** The color $\mathbf{c}_i$ is retrieved by evaluating the spherical harmonics at the viewing direction $\mathbf{v}$ (from camera center to $\boldsymbol{\mu}_i$) [S6][S12].
    *   **Accumulation:** The pixel color $\mathbf{C}$ is computed via alpha compositing:
        $$ \mathbf{C} = \sum_{i \in \mathcal{N}} \mathbf{c}_i \alpha'_i \prod_{j=1}^{i-1} (1 - \alpha'_j) $$
        where $\mathcal{N}$ is the set of sorted Gaussians for the pixel, and the product term represents the accumulated transmittance $T_i$ [S1, S6][S16, S23, unofficial].

**Training Loop:**
*   **Loss Function:** The rendered image $\tilde{C}$ is compared to the ground truth image $C$ using a weighted combination of L1 loss and D-SSIM (Difference SSIM):
    $$ \mathcal{L} = (1 - \lambda) \mathcal{L}_{L1} + \lambda \mathcal{L}_{D-SSIM} $$
    where $\lambda$ is a hyperparameter [S6][S13][S35, unofficial].
*   **Optimization:** Parameters $\boldsymbol{\mu}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i$ are updated via stochastic gradient descent [S6][S35, unofficial].
*   **Adaptive Density Control:** To manage the number of Gaussians, the method periodically (e.g., every 100 iterations) performs:
    1.  **Clone:** Small Gaussians with large position gradients are cloned to under-reconstructed regions [S1][S19][S35, unofficial].
    2.  **Split:** Large Gaussians with large position gradients are split into two smaller Gaussians to over-reconstructed regions [S1][S19][S35, unofficial].
    3.  **Prune:** Gaussians with opacity below a threshold $\tau$ or screen-space size exceeding a limit are removed [S1][S19].
    4.  **Opacity Reset:** Opacities are periodically reset to a small value to prune redundancies [S1][S19].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class Tile:
    def __init__(self, x_start, y_start, x_end, y_end):
        self.x_start, self.y_start = x_start, y_start
        self.x_end, self.y_end = x_end, y_end

def create_tiles(image_width, image_height, tile_size):
    tiles = []
    for y in range(0, image_height, tile_size):
        for x in range(0, image_width, tile_size):
            tiles.append(Tile(x, y, min(x + tile_size, image_width), min(y + tile_size, image_height)))
    return tiles

def quaternion_to_rotation_matrix(q):
    w, x, y, z = q
    return np.array([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
        [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
        [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
    ])

def diag_matrix(s):
    return np.diag(s)

def inverse_2x2(M):
    det = M[0,0]*M[1,1] - M[0,1]*M[1,0]
    return np.array([[M[1,1]/det, -M[0,1]/det], [-M[1,0]/det, M[0,0]/det]])

def compute_bounding_box(mu_2d, Sigma_2d):
    eigenvalues, eigenvectors = np.linalg.eigh(Sigma_2d)
    std_devs = 2.0 * np.sqrt(eigenvalues)
    dx = std_devs[0] + std_devs[1]
    dy = std_devs[0] + std_devs[1]
    return (mu_2d[0] - dx, mu_2d[1] - dy, mu_2d[0] + dx, mu_2d[1] + dy)

def intersects_bbox(bbox, tile):
    return not (bbox[2] < tile.x_start or bbox[0] > tile.x_end or bbox[3] < tile.y_start or bbox[1] > tile.y_end)

def evaluate_sh_basis(theta, phi, degree):
    basis = np.zeros((degree + 1)**2)
    basis[0] = 0.28209479177387814
    if degree >= 1:
        basis[1] = 0.4886025119029199 * np.sin(theta) * np.cos(phi)
        basis[2] = 0.4886025119029199 * np.sin(theta) * np.sin(phi)
        basis[3] = 0.4886025119029199 * np.cos(theta)
    if degree >= 2:
        basis[4] = 1.0925484305920792 * np.sin(theta)**2 * np.cos(2*phi)
        basis[5] = 1.0925484305920792 * np.sin(theta)**2 * np.sin(2*phi)
        basis[6] = 0.31539156525252005 * (3*np.cos(theta)**2 - 1)
        basis[7] = 1.0925484305920792 * np.sin(theta)**2 * np.cos(2*phi)
        basis[8] = 0.4886025119029199 * np.sin(theta) * np.cos(phi)
        basis[9] = 0.5462742152960396 * np.sin(theta)**2 * np.sin(2*phi)
        basis[10] = 0.5462742152960396 * np.sin(theta) * np.cos(phi)
        basis[11] = 0.5462742152960396 * np.sin(theta) * np.sin(phi)
        basis[12] = 0.9461746957575601 * np.cos(theta)**2 - 0.31539156525252005
        basis[13] = 0.5462742152960396 * np.sin(theta) * np.cos(phi)
        basis[14] = 0.5462742152960396 * np.sin(theta) * np.sin(phi)
        basis[15] = 0.10578554698323738 * (5*np.cos(theta)**2 - 3)
    return basis

def evaluate_sh(sh_coeffs, view_dir):
    theta = np.arccos(np.clip(view_dir[2], -1.0, 1.0))
    phi = np.arctan2(view_dir[1], view_dir[0])
    sh_basis = evaluate_sh_basis(theta, phi, degree=3)
    color = np.zeros(3)
    for c in range(3):
        color[c] = np.dot(sh_coeffs[c], sh_basis)
    return np.clip(color, 0, 1)

def compute_jacobian(mu_cam, K):
    x, y, z = mu_cam
    fx, fy = K[0, 0], K[1, 1]
    cx, cy = K[0, 2], K[1, 2]
    J = np.zeros((2, 3))
    J[0, 0] = fx / z
    J[0, 2] = -fx * x / (z**2)
    J[1, 1] = fy / z
    J[1, 2] = -fy * y / (z**2)
    return J

def project_point(mu_cam, K):
    x, y, z = mu_cam
    fx, fy = K[0, 0], K[1, 1]
    cx, cy = K[0, 2], K[1, 2]
    u = fx * x / z + cx
    v = fy * y / z + cy
    return np.array([u, v])

def render_gaussians(gaussians, camera_pose, intrinsics, image_width, image_height):
    processed_gaussians = []
    for g in gaussians:
        R = quaternion_to_rotation_matrix(g.q)
        S = diag_matrix(g.s)
        Sigma = R @ S @ S.T @ R.T
        mu_cam = camera_pose[:3, :3] @ g.mu + camera_pose[:3, 3]
        J = compute_jacobian(mu_cam, intrinsics)
        W_rot = camera_pose[:3, :3]
        Sigma_2d = J @ W_rot @ Sigma @ W_rot.T @ J.T
        mu_2d = project_point(mu_cam, intrinsics)
        Sigma_2d_inv = inverse_2x2(Sigma_2d)
        bbox = compute_bounding_box(mu_2d, Sigma_2d)
        processed_gaussians.append({
            'mu_2d': mu_2d,
            'conic': Sigma_2d_inv,
            'alpha': g.alpha,
            'sh_coeffs': g.sh_coeffs,
            'bbox': bbox,
            'depth': mu_cam[2]
        })
    
    tile_size = 16
    tiles = create_tiles(image_width, image_height, tile_size)
    image = np.zeros((image_height, image_width, 3))
    
    for tile in tiles:
        tile_gaussians = [g for g in processed_gaussians if intersects_bbox(g['bbox'], tile)]
        tile_gaussians.sort(key=lambda g: g['depth'], reverse=True)
        
        for y in range(tile.y_start, min(tile.y_end, image_height)):
            for x in range(tile.x_start, min(tile.x_end, image_width)):
                pixel_color = np.array([0.0, 0.0, 0.0])
                transmittance = 1.0
                
                for g in tile_gaussians:
                    dx = x - g['mu_2d'][0]
                    dy = y - g['mu_2d'][1]
                    C00, C01, C11 = g['conic']
                    power = -0.5 * (C00 * dx**2 + C11 * dy**2 + 2 * C01 * dx * dy)
                    power = max(power, -2.0)
                    gaussian_val = np.exp(power)
                    alpha_eff = g['alpha'] * gaussian_val
                    view_dir = np.array([g['mu_2d'][0], g['mu_2d'][1], 1.0])
                    view_dir = view_dir / np.linalg.norm(view_dir)
                    color = evaluate_sh(g['sh_coeffs'], view_dir)
                    pixel_color += color * alpha_eff * transmittance
                    transmittance *= (1.0 - alpha_eff)
                    if transmittance < 1e-4:
                        break
                image[y, x] = pixel_color
    return image
```

**Edge Cases and Pitfalls:**
*   **Precision:** The exponential function in the 2D Gaussian evaluation can underflow to zero for distant pixels. Clamping the power term to a minimum value (e.g., -2.0) prevents numerical instability [S20].
*   **Boundaries:** Gaussians near the camera plane can cause division by zero in the Jacobian computation. A small epsilon should be added to the depth $z$ in the Jacobian calculation [G].
*   **Degenerate Input:** If the SfM point cloud is sparse or noisy, initialization may fail. Random initialization can be used but converges slower [S19].
*   **Covariance Stability:** Directly optimizing the covariance matrix can lead to non-positive semi-definite matrices. Using quaternion and scale parameterization ensures validity [S6][S25].

## Subtopics

### [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md)
The Unnormalized 3D Gaussian Function is a Gaussian representation used in 3D Gaussian Splatting where the amplitude $a$ at the mean $\boldsymbol{\mu}$ equals the opacity, representing the peak capacity to block or emit light. It differs from the [Normalized 3D Gaussian Function](1064-normalized-3d-gaussian-function.md) by parameterizing the function via its peak value rather than its integral, making $a$ the intensity at the center rather than the total light capacity.

### [Row-constant hoisting for Gaussian Splatting Rasterization](1195-row-constant-hoisting-for-gaussian-splatting-rasterization.md)
Row-constant hoisting is an optimization for the inner pixel loop of a Gaussian splatting rasterizer that hoists row-constant terms outside the inner (column) loop to minimize arithmetic operations per pixel. It reduces the computational cost of evaluating the 2D Gaussian exponent by precomputing terms dependent only on the row coordinate, allowing the inner loop to perform fewer multiplications and additions per column. This method is a specific implementation detail within the broader [Gaussian Splatting](946-gaussian-splatting.md) rendering pipeline, distinct from higher-level optimizations like [Contribution-based ordering](8748-contribution-based-ordering.md) or [RoofGS Dual-Pixel Incremental Evaluation](9827-roofgs-dual-pixel-incremental-evaluation.md).

### [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md)
This method projects a 3D Gaussian primitive, defined by a mean and a covariance matrix in world space, into a 2D elliptical footprint (splat) on the image plane for rasterization. It solves the problem of rendering millions of anisotropic 3D Gaussians in real-time by avoiding ray marching; instead, it approximates the non-linear perspective projection with a local affine transformation, preserving the Gaussian form in 2D. It sets itself apart from [Optimal Gaussian Splatting Projection](3311-optimal-gaussian-splatting-projection.md) by using a fixed image plane rather than a per-Gaussian tangent plane, and from [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md) by adding a constant epsilon to the diagonal for positive definiteness rather than rescaling the determinant.

### [Gaussian in the Wild (GS-W)](1386-gaussian-in-the-wild-gs-w.md)
Gaussian in the Wild (GS-W) is a novel view synthesis method for unconstrained image collections that explicitly separates scene appearance into intrinsic material properties and dynamic environmental factors (such as illumination and weather) per 3D Gaussian point. It differs from standard 3D Gaussian Splatting and NeRF-based methods by avoiding global latent embeddings, which allows it to capture local high-frequency changes without blurring intrinsic textures with dynamic variations. The method employs adaptive sampling from image features and 2D visibility maps to mitigate transient occlusions and photometric variations.

### [3DGUT](1474-3dgut.md)
3DGUT is a Gaussian projection technique that employs an unscented transform to improve the affine approximation of the projection operator, providing more precise Gaussian projection onto the image plane. It replaces the standard first-order Taylor expansion used in Gaussian Splatting with a non-linear propagation of statistical samples to reduce projection approximation errors.

### [Transformer-based Reconstruction](1794-transformer-based-reconstruction.md)
Transformer-based Reconstruction methods replace the iterative optimization loop of standard 3D Gaussian Splatting (3DGS) with a feed-forward neural network that directly predicts Gaussian parameters from input images. This approach solves the computational bottleneck of per-scene optimization, enabling real-time reconstruction from single or sparse views. It differs from iterative siblings by eliminating the need for gradient descent on scene parameters, instead relying on learned priors within the transformer architecture to generalize across scenes.

### [3D Gaussian Ellipsoid Rendering](1869-3d-gaussian-ellipsoid-rendering.md)
3D Gaussian Ellipsoid Rendering (VoGE) is a differentiable volume rendering method that represents scenes as a sum of 3D Gaussian ellipsoids and computes image synthesis via ray tracing rather than rasterization. It solves the occlusion ambiguity inherent in distance-based blending by accumulating volume density distributions along viewing rays, enabling realistic interaction between overlapping objects. Unlike its sibling [Gaussian Splatting](946-gaussian-splatting.md), which approximates volume rendering through 2D rasterization of projected splats, VoGE replaces rasterization with ray tracing to capture exact volume density distributions for blending.

### [Stochastic Gaussian Splatting](6599-stochastic-gaussian-splatting.md)
Stochastic Gaussian Splatting (SGS) refers to two distinct methodological extensions of 3D Gaussian Splatting that replace deterministic rasterization with probabilistic sampling. The first variant, often termed **Bayesian SGS**, treats Gaussian parameters as random variables to enable uncertainty estimation alongside novel view synthesis. The second variant, **Gaussian Point Splatting**, eliminates the per-tile radix sort and tile-based rendering bottlenecks by sampling opaque points from Gaussians and compositing them via atomic operations, enabling the interactive rendering of hundreds of millions of Gaussians. Both methods rely on Variational Inference or stochastic transparency to approximate the volume rendering integral without explicit depth sorting.

### [Rendering Implementation](7155-rendering-implementation.md)
Rendering Implementation defines the computational pipeline that transforms a set of 3D Gaussian primitives into a 2D image via screen-space rasterization and alpha compositing . It solves the real-time rendering bottleneck of volumetric methods by replacing ray marching with a differentiable rasterization pipeline, allowing gradients to flow from the rendered image back to Gaussian parameters . It sets itself apart from its siblings by relying on full 3D covariance projection and tile-based depth sorting to handle arbitrary 3D geometry and occlusion .

### [Contribution-based ordering](8748-contribution-based-ordering.md)
Contribution-based ordering is a post-training method for 3D Gaussian Splatting (3DGS) that sorts primitives by their visual importance to enable progressive rendering. It solves the problem of inefficient data streaming by ensuring high-impact splats are transmitted and rendered first. Unlike [EAGLES](1386-gaussian-in-the-wild-gs-w.md) which uses this metric during training to prune splats, this method operates on the fully trained model to select impactful splats for independent streaming.

### [3D Convex Splatting](8758-3d-convex-splatting.md)
3D Convex Splatting is a neural rendering method that replaces standard 3D Gaussian primitives with smooth convex shapes to represent scenes. It solves the limitation of Gaussians in accurately modeling flat surfaces and sharp edges, offering greater geometric flexibility and requiring fewer primitives for equivalent fidelity. Unlike its parent [Gaussian Splatting](946-gaussian-splatting.md), which relies on elliptical distributions, this method introduces non-elliptical convex boundaries to better approximate complex geometry.

### [Feed-forward 3D Gaussian Splatting](9932-feed-forward-3d-gaussian-splatting.md)
Feed-forward 3D Gaussian Splatting is a method that predicts a complete 3D Gaussian representation of a scene directly from a sparse set of input images, bypassing the iterative optimization required by standard [Gaussian Splatting](946-gaussian-splatting.md). It solves the problem of slow reconstruction by using a multi-view Transformer to map image features to Gaussian primitives in a single forward pass. It differs from iterative methods by trading per-scene training time for inference latency, and from other feed-forward approaches by explicitly predicting the full set of Gaussian attributes (position, covariance, opacity, color) via a linear projection of transformer latents.

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N \cdot T)$, where $N$ is the number of Gaussians and $T$ is the number of tiles, but effectively $O(P \cdot K)$ per pixel, where $P$ is the number of pixels and $K$ is the average number of overlapping Gaussians per tile. Sorting is $O(N \log N)$ [S6][S16, unofficial].
*   **Memory Complexity:** $O(N)$, storing parameters for each Gaussian. Typical scenes require millions of Gaussians, leading to high memory usage [S5, S21].
*   **Measured Performance:** Real-time rendering (60+ FPS) is achievable on modern GPUs for scenes with hundreds of thousands to millions of Gaussians [S1, S6]. Training time is significantly faster than NeRFs (hours vs. days) but slower than feed-forward methods [S5].
*   **Strengths:** Real-time rendering, high visual fidelity, explicit representation allows for easy editing and integration with graphics pipelines [S1, S6][S12].
*   **Limitations:** High memory consumption and potential for popping artifacts due to depth sorting approximations [S2, S5].
*   **When to Use:** Real-time novel view synthesis, interactive applications, and scenarios requiring high-quality rendering from sparse inputs [S1, S5].
*   **When Not to Use:** Memory-constrained environments, scenes requiring precise volumetric effects (e.g., smoke, fire) without extensions, or real-time applications on low-end hardware [S2, S5].

## Background
Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1][S12]. It builds on earlier work in volume splatting, particularly Elliptical Weighted Average (EWA) splatting [S2][S4, unofficial], and point-based rendering methods [S2]. The method was developed to address the computational limitations of NeRFs, offering a hybrid approach that combines the optimization benefits of neural networks with the efficiency of explicit rasterization [S6][S12]. It has since become a dominant paradigm in 3D scene representation and novel view synthesis [S5].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S3] SparSplat: Fast Multi-View Reconstruction with Generalizable 2D Gaussian Splatting, https://doi.org/10.48550/arXiv.2505.02175, fetched 2026-10-26 (unofficial)
[S4] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S5] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S6] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S7] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S8] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26
[S9] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27
[S10] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26 (unofficial)
[S11] Refining Gaussian Splatting: A Volumetric Densification Approach, https://doi.org/10.24132/CSRN.2025-6, fetched 2026-10-26 (unofficial)
[S12] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26
[S13] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26
[S14] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26
[S15] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26 (unofficial)
[S16] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26 (unofficial)
[S17] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26 (unofficial)
[S18] UltraGauss: Ultrafast Gaussian Reconstruction of 3D Ultrasound Volumes, https://doi.org/10.48550/arXiv.2505.05643, fetched 2026-10-27 (unofficial)
[S19] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27
[S20] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27
[S21] Neural Rendering: Graphics Learned From Images, https://www.ultralytics.com/glossary/neural-rendering, fetched 2026-10-27
[S22] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S23] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27 (unofficial)
[S24] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28 (unofficial)
[S25] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30
[S26] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S27] PDF Lecture 15: Rendering Volumes, Points, and Gaussians, https://gfxcourses.stanford.edu/cs248a/winter25content/media/volumespoints/15_volumespoints.pdf, fetched 2026-10-28
[S28] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S29] FreeSplat: Generalizable 3D Gaussian Splatting Towards Free-View Synthesis of Indoor Scenes, https://doi.org/10.48550/arXiv.2405.17958, fetched 2026-10-29 (unofficial)
[S30] No Redundancy, No Stall: Lightweight Streaming 3D Gaussian Splatting for Real-time Rendering, https://doi.org/10.1109/iccad66269.2025.11240950, fetched 2026-10-30 (unofficial)
[S31] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
[S32] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29
[S33] Differentiable Rendering - 可微渲染：从2D 学习3D, https://www.ultralytics.com/zh/glossary/differentiable-rendering, fetched 2026-10-30
[S34] 从体渲染到3D Gaussian Splatting：理论与应用, https://www.alphaxiv.org/zh/abs/2510.18101, fetched 2026-10-30
[S35] Gaussian Splatting - 高斯泼溅：实时3D 渲染, https://www.ultralytics.com/zh/glossary/gaussian-splatting, fetched 2026-10-30 (unofficial)
[S36] 众趣分享丨浅谈空间计算技术基础---3D 高斯泼溅, https://www.3dnest.cn/article/1280.html, fetched 2026-10-30 (unofficial)
