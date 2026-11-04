# 3D Gaussian Splatting (3DGS) Preliminaries

Also called: Kerbl et al. 3DGS, Standard 3DGS representation, Vanilla 3D Gaussian Splatting

## Overview
3D Gaussian Splatting (3DGS) is a differentiable, point-based scene representation that replaces the implicit multi-layer perceptron (MLP) and ray-marched volume integration of NeRF with an explicit collection of 3D anisotropic Gaussians [S3, unofficial]. It enables real-time synthesis of high-fidelity novel views via a GPU-friendly tile-based rasterizer that depth-sorts and alpha-composites 2D projections of these Gaussians [S2, unofficial][S5]. Unlike NeRF, which infers density and color at arbitrary points, 3DGS optimizes the parameters of a fixed set of Gaussian primitives (position, covariance, opacity, and spherical harmonic coefficients) to minimize photometric loss, using adaptive density control to adjust the number of primitives during training [S3, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $N$ 3D Gaussian primitives, each defined by a mean $\boldsymbol{\mu} \in \mathbb{R}^3$, a covariance matrix $\Sigma \in \mathbb{R}^{3\times3}$, an opacity scalar $\alpha$, and spherical harmonic (SH) coefficients for view-dependent color [S2, S3, unofficial].
*   A camera view defined by world-to-camera matrix $W$ and projection parameters [S3, unofficial].
*   Training images $I_v$ for photometric supervision [S3, unofficial].

**Outputs:**
*   Rendered image $\hat{I}_v$ for a given camera view [S3, unofficial].
*   Optimized Gaussian parameters after training [S3, unofficial].

### Data Structures
Each Gaussian primitive $i$ stores the following parameters, which are optimized via gradient descent [S3, unofficial][S4]:
*   **Mean:** $\boldsymbol{\mu}_i \in \mathbb{R}^3$ (3D position).
*   **Rotation:** A unit quaternion $q_i \in \mathbb{R}^4$ representing orientation, normalized after each optimization step [S3, unofficial][S4].
*   **Scale:** A vector $\mathbf{s}_i \in \mathbb{R}^3$ representing scale along principal axes, stored in logarithmic space to ensure positivity [S3, unofficial][S4].
*   **Opacity:** A raw logit $\alpha_i \in \mathbb{R}$, mapped to $[0,1]$ via a Sigmoid activation during rendering [S3, unofficial][S4].
*   **Color:** Spherical Harmonic coefficients up to degree $\ell=3$, resulting in 16 coefficients per channel (48 floats total per Gaussian) [S3, unofficial][S4].

The covariance matrix $\Sigma_i$ is constructed from the rotation and scale parameters [S2, S3, unofficial]:
$$ \Sigma_i = R_i \, \mathrm{diag}(\mathbf{s}_i)^2 \, R_i^\top $$
where $R_i$ is the rotation matrix derived from the unit quaternion $q_i$ [S3, unofficial].

### Rendering Pipeline

1.  **Culling:** Gaussians outside the view frustum or behind the camera are discarded [S3, unofficial].
2.  **Projection:** Each Gaussian is projected into screen space. The 3D covariance $\Sigma$ is transformed to a 2D covariance $\Sigma_{2D}$ using the Jacobian $J$ of the perspective projection at the projected center $\boldsymbol{\mu}_c = W\boldsymbol{\mu}$ [S3, unofficial]:
    $$ \Sigma_{2D} = J \, W \, \Sigma \, W^\top J^\top \in \mathbb{R}^{2\times2} $$
    This is an affine approximation (EWA splatting) [S3, unofficial].
3.  **Bounding Box:** A 2D axis-aligned bounding box (AABB) is computed for each Gaussian based on $\Sigma_{2D}$, typically expanded to cover the $3\sigma$ influence region ($d^\top \Sigma^{-1} d \leq 9$) [S2, S3, unofficial][S4].
4.  **Tiling:** The image is divided into tiles (default $16\times16$ pixels) [S3, unofficial]. Gaussians are binned into tile lists based on their AABB overlap [S3, unofficial].
5.  **Sorting:** Within each tile, Gaussians are sorted front-to-back by their depth (center depth) [S3, unofficial].
6.  **Rasterization and Compositing:** For each pixel, the 2D Gaussian value is computed and alpha-composited with previously accumulated color [S2, S3, unofficial].

### Formulas

**3D Gaussian Function:**
The unnormalized Gaussian function at position $\mathbf{x}$ is [S2, S3, unofficial][S5]:
$$ G(\mathbf{x}) = \exp\!\left(-\tfrac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^\top \Sigma^{-1}(\mathbf{x}-\boldsymbol{\mu})\right) $$

**2D Gaussian Evaluation:**
For a pixel $\mathbf{p}$, the 2D Gaussian footprint $G_i^{2D}(\mathbf{p})$ is evaluated using the projected covariance $\Sigma_{2D}$ and the 2D displacement from the projected center [S3, unofficial].

**Alpha Compositing:**
The color $C(\mathbf{p})$ at pixel $\mathbf{p}$ is computed by front-to-back alpha compositing of the sorted Gaussians $N$ contributing to that pixel [S2, S3, unofficial]:
$$ C(\mathbf{p}) = \sum_{i\in N} \mathbf{c}_i(\mathbf{d}) \, \alpha_i \, G_i^{2D}(\mathbf{p}) \prod_{j<i}\!\left(1 - \alpha_j \, G_j^{2D}(\mathbf{p})\right) $$
where $\mathbf{c}_i(\mathbf{d})$ is the view-dependent color evaluated from the SH coefficients for viewing direction $\mathbf{d}$ [S3, unofficial].

**Loss Function:**
Training minimizes a combination of L1 loss and SSIM loss [S3, unofficial]:
$$ \mathcal{L} = (1-\lambda)\,\|\hat{I}_v - I_v\|_1 + \lambda\,(1 - \mathrm{SSIM}(\hat{I}_v, I_v)) $$
with $\lambda = 0.2$ [S3, unofficial].

### Training Loop and Adaptive Density Control

1.  **Initialization:** Start from a Structure-from-Motion (SfM) sparse point cloud (e.g., COLMAP). If no SfM is available, use random uniform points in the scene bound [S3, unofficial].
2.  **Optimization:** Each iteration, pick a random training view, render $\hat{I}_v$, compute loss, and backpropagate through the rasterizer to update parameters $\{\boldsymbol{\mu}, q, s, \alpha, \mathrm{SH}\}$ using Adam [S3, unofficial].
3.  **Adaptive Density Control (ADC):** Applied every $K$ iterations (typically every 100 iterations within a window of 500–15,000) [S3, unofficial]:
    *   **Densification:** Where the 2D position gradient magnitude $\|\nabla_{\boldsymbol{\mu}_{2D}} L\|$ exceeds a threshold $\tau_\nabla \sim 2\times10^{-4}$ [S3, unofficial]:
        *   **Clone:** For small Gaussians (under-reconstructed), duplicate and perturb position [S3, unofficial].
        *   **Split:** For large Gaussians (over-reconstructed), replace with two smaller Gaussians [S3, unofficial].
    *   **Pruning:** Remove Gaussians with opacity below $\epsilon_\alpha \sim 0.005$, or those that are off-screen or have huge projections [S3, unofficial].
    *   **Opacity Reset:** Every 3000 iterations, reset opacity to $0.01$ to escape local minima [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_gaussians(gaussians, camera_matrix, image_shape, tile_size=16):
    """
    Renders a set of 3D Gaussians to an image using tile-based rasterization.
    """
    H, W = image_shape
    num_tiles_h = (H + tile_size - 1) // tile_size
    num_tiles_w = (W + tile_size - 1) // tile_size
    
    # Initialize output buffers
    C = np.zeros((H, W, 3), dtype=np.float32)
    T = np.zeros((H, W), dtype=np.float32)
    
    # Precompute 2D projections and bounding boxes
    projected_gaussians = []
    for g in gaussians:
        mu_cam = camera_matrix[:3, :3] @ g.mu + camera_matrix[:3, 3]
        if mu_cam[2] <= 0.1:
            continue
            
        R = quaternion_to_rotation_matrix(g.q)
        S = np.diag(np.exp(g.s))
        Sigma_3D = R @ (S @ S.T) @ R.T
        
        z = mu_cam[2]
        fx, fy = 1.0, 1.0
        cx, cy = W / 2.0, H / 2.0
        
        J = np.array([
            [fx/z, 0,    -fx*mu_cam[0]/(z*z)],
            [0,    fy/z, -fy*mu_cam[1]/(z*z)],
            [0,    0,    0]
        ])
        
        W_rot = camera_matrix[:3, :3]
        Sigma_2D = J @ W_rot @ Sigma_3D @ W_rot.T @ J.T
        
        eigenvalues, _ = np.linalg.eigh(Sigma_2D)
        max_extent = 3.0 * np.sqrt(np.maximum(eigenvalues, 1e-6))
        
        x_2d = fx * mu_cam[0] / mu_cam[2] + cx
        y_2d = fy * mu_cam[1] / mu_cam[2] + cy
        
        min_x = int(max(0, x_2d - max_extent[0]))
        max_x = int(min(W, x_2d + max_extent[0]))
        min_y = int(max(0, y_2d - max_extent[1]))
        max_y = int(min(H, y_2d + max_extent[1]))
        
        if min_x >= max_x or min_y >= max_y:
            continue
            
        projected_gaussians.append({
            'mu_2d': np.array([x_2d, y_2d]),
            'Sigma_2D': Sigma_2D,
            'alpha': 1.0 / (1.0 + np.exp(-g.alpha)),
            'sh_coeffs': g.sh_coeffs,
            'depth': mu_cam[2],
            'bbox': (min_x, max_x, min_y, max_y)
        })
        
    # Bin into tiles
    tiles = {}
    for i, pg in enumerate(projected_gaussians):
        min_x, max_x, min_y, max_y = pg['bbox']
        tile_min_x = min_x // tile_size
        tile_max_x = (max_x - 1) // tile_size
        tile_min_y = min_y // tile_size
        tile_max_y = (max_y - 1) // tile_size
        
        for ty in range(tile_min_y, tile_max_y + 1):
            for tx in range(tile_min_x, tile_max_x + 1):
                key = (ty, tx)
                if key not in tiles:
                    tiles[key] = []
                tiles[key].append((i, pg['depth']))
                
    # Sort and Composite per tile
    for (ty, tx), tile_data in tiles.items():
        tile_data.sort(key=lambda x: x[1])
        y_start = ty * tile_size
        x_start = tx * tile_size
        
        for idx, depth in tile_data:
            pg = projected_gaussians[idx]
            mu_2d = pg['mu_2d']
            Sigma_2D = pg['Sigma_2D']
            alpha = pg['alpha']
            sh_coeffs = pg['sh_coeffs']
            
            min_x, max_x, min_y, max_y = pg['bbox']
            min_x = max(min_x, x_start)
            max_x = min(max_x, x_start + tile_size)
            min_y = max(min_y, y_start)
            max_y = min(max_y, y_start + tile_size)
            
            ys, xs = np.meshgrid(np.arange(min_y, max_y), np.arange(min_x, max_x), indexing='ij')
            pixel_coords = np.stack([xs.flatten(), ys.flatten()], axis=-1).astype(np.float32)
            
            diff = pixel_coords - mu_2d
            det = Sigma_2D[0,0]*Sigma_2D[1,1] - Sigma_2D[0,1]*Sigma_2D[1,0]
            if abs(det) < 1e-6:
                continue
            inv_Sigma_2D = np.array([
                [Sigma_2D[1,1], -Sigma_2D[0,1]],
                [-Sigma_2D[1,0], Sigma_2D[0,0]]
            ]) / det
            
            exponents = -0.5 * np.sum((diff @ inv_Sigma_2D) * diff, axis=1)
            g_2d_values = np.exp(exponents)
            
            d = np.array([0, 0, 1])
            color = evaluate_sh(sh_coeffs, d)
            
            flat_g_2d = g_2d_values.reshape(-1)
            flat_color = np.tile(color, (len(flat_g_2d), 1))
            
            pixel_indices = np.arange(len(flat_g_2d))
            img_y = min_y + (pixel_indices // (max_x - min_x))
            img_x = min_x + (pixel_indices % (max_x - min_x))
            
            for k in range(len(flat_g_2d)):
                y, x = img_y[k], img_x[k]
                a = alpha * flat_g_2d[k]
                if a < 1e-4:
                    continue
                t_prev = T[y, x]
                if t_prev > 0.9999:
                    continue
                C[y, x] += flat_color[k] * a * (1 - t_prev)
                T[y, x] += a * (1 - t_prev)
                
    return C

def evaluate_sh(coeffs, direction):
    """Evaluate spherical harmonics for a given direction."""
    return coeffs[:3]

def quaternion_to_rotation_matrix(q):
    """Convert unit quaternion to 3x3 rotation matrix."""
    w, x, y, z = q
    return np.array([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
        [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
        [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
    ])
```

### Parameters and Hyperparameters
*   **Iterations:** 30,000 (7k–15k for fast preview) [S3, unofficial].
*   **Learning Rates:** Position: $1.6\times10^{-4}$ (scaled by scene extent, exponential decay) [S3, unofficial]. Opacity: $5\times10^{-2}$ (sigmoid raw) [S3, unofficial]. Scale: $5\times10^{-3}$ (exp raw) [S3, unofficial]. Rotation: $10^{-3}$ (quaternion, normalized after step) [S3, unofficial]. SH: $2.5\times10^{-3}$ for DC, $/20$ for higher bands [S3, unofficial].
*   **Densification:** Gradient threshold $\tau_\nabla \sim 2\times10^{-4}$, applied every 100 iterations within a window of 500–15,000 [S3, unofficial].
*   **Opacity Reset:** Every 3000 iterations, set $\alpha \leftarrow 0.01$ [S3, unofficial].
*   **Tile Size:** $16\times16$ [S3, unofficial].
*   **SSIM Weight:** $\lambda = 0.2$ [S3, unofficial].
*   **SH Warm-up:** Start with degree 0, add 1 band every 1000 iterations up to degree 3 [S3, unofficial].

### Edge Cases and Pitfalls
*   **Popping Artifacts:** The per-tile depth sort uses the Gaussian center depth, not per-pixel depth. In distant tiles with strong overlap, the sort order can flip across views, causing popping [S3, unofficial].
*   **Pre-classification Limitation:** Color is assigned prior to compositing (pre-classification), which is not aligned with transfer function mapping in alpha-blended ray casting, leading to inaccurate TF evaluation and artifacts in overlapping regions [S2, unofficial].
*   **Heuristic Densification:** Adaptive density control based on 2D projections may fail to capture local structural characteristics, generating excessive overlapping Gaussians [S2, unofficial].
*   **3σ Support:** The $3\sigma$ cutoff ($d^\top \Sigma^{-1} d \leq 9$) often overestimates the primitive footprint, leading to redundant primitives and increased memory cost [S2, unofficial].

## Subtopics

### [3D Gaussian Splatting Optimization Parameters](3460-3d-gaussian-splatting-optimization-parameters.md)
Details the specific learning rates, decay schedules, and hyperparameters used in the Adam optimizer for 3DGS.

### [ECoNGS Optimization and Coding](2087-econgs-optimization-and-coding.md)
A variant focusing on efficient coding and optimization strategies for 3DGS.

### [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md)
Describes the data structure for storing and initializing the list of Gaussians.

### [Gaussian Property Loading](2221-gaussian-property-loading.md)
Methods for loading Gaussian properties from various sources.

### [3D Covariance Computation](2231-3d-covariance-computation.md)
Algorithms for computing the 3D covariance matrix from rotation and scale parameters.

### [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md)
Memory layout optimizations for 3D Gaussian data.

### [N-DGS Bounded Cholesky Parameterization](2811-n-dgs-bounded-cholesky-parameterization.md)
An alternative parameterization for covariance matrices using bounded Cholesky decomposition.

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ per pixel where $N$ is the number of Gaussians contributing to the pixel, but accelerated by tiling and culling [G]. Training is dominated by the rasterization and backpropagation steps, scaling with the number of Gaussians and image resolution [S3, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of Gaussians [G]. Each Gaussian stores $\boldsymbol{\mu}$ (12 bytes), $q$ (16 bytes), $s$ (12 bytes), $\alpha$ (4 bytes), and SH coefficients (192 bytes for degree 3), totaling ~236 bytes per Gaussian [G].
*   **Performance:** Real-time rendering (100–300 FPS at 1080p) is achievable on modern GPUs due to the tile-based rasterizer [S3, unofficial].
*   **Strengths:** High-fidelity rendering, real-time performance, explicit representation allows for easy editing and manipulation [G].
*   **Limitations:** Popping artifacts due to depth sorting, pre-classification artifacts, and heuristic densification can lead to redundancy [S2, S3, unofficial]. Not inherently designed for volumetric media, leading to inaccuracies in TF evaluation [S2, unofficial].
*   **When to Use:** Ideal for static scene reconstruction and novel view synthesis where real-time performance and high visual quality are required [G].
*   **When Not to Use:** Not suitable for scenes with complex volumetric effects (e.g., smoke, fire) without modifications, as it does not model volumetric ray integration [S2, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 as a state-of-the-art reconstruction approach [S3, unofficial]. It represents scenes using 3D Gaussians, serving as a baseline for many subsequent methods despite not being specifically designed for volumetric media [S1]. The method leverages hardware rasterization for efficient rendering, replacing the implicit MLPs of NeRF with explicit Gaussian primitives [S2, S3, unofficial]. It has become a foundational technique in computer graphics, inspiring numerous variants for dynamic scenes, compression, and volumetric rendering [S5].

## Sources
[S1] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
[S2] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] Efficient Perspective‐Correct 3D Gaussian Splatting Using Hybrid Transparency, https://doi.org/10.1111/cgf.70014, fetched 2026-10-30
[S5] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30
