# 3D Gaussian Splatting (3DGS) Preliminaries

Also called: 3D Gaussian Splatting (3D-GS), 3D Gaussian Splatting (3DGS) Representation, 3D-GS, Standard 3DGS representation, Vanilla 3D Gaussian Splatting

## Overview
3D Gaussian Splatting (3DGS) is a scene representation method that replaces implicit neural fields with an explicit set of anisotropic 3D Gaussian primitives, enabling real-time rendering via GPU rasterization without ray marching [S1, S2][S5, S7, S9, unofficial]. It solves the trade-off between the high visual quality of neural radiance fields (NeRFs) and the speed of explicit geometry by representing the scene as millions of splats (ellipsoids) that are projected and alpha-composited [S5, S6, unofficial][S8]. Unlike NeRFs, which require per-pixel network queries and volume integration, 3DGS uses a differentiable sort-and-blend rasterizer, allowing for end-to-end optimization of geometric and photometric attributes [S9, S12, unofficial]. It sets itself apart from its siblings by serving as the foundational "vanilla" baseline that uses fixed-degree spherical harmonics and heuristic adaptive density control, whereas variants like [ECoNGS Optimization and Coding](2087-econgs-optimization-and-coding.md) focus on compression or [N-DGS Bounded Cholesky Parameterization](2811-n-dgs-bounded-cholesky-parameterization.md) focus on parameterization stability.

## How it works
**Inputs and Outputs**
*   **Inputs:** A sparse point cloud (typically from Structure-from-Motion, e.g., COLMAP) providing initial positions, and a set of $N$ training images with known camera poses $P$.
*   **Outputs:** A rendered image $\hat{I}$ for a novel view, and a set of optimized 3D Gaussian parameters $\{\mu_k, q_k, s_k, \alpha_k, \text{SH}_k\}_{k=1}^K$.

**Data Structures**
Each Gaussian primitive $k$ stores the following parameters, which are optimized via gradient descent:
1.  **Mean Position:** $\boldsymbol{\mu}_k \in \mathbb{R}^3$ [S2][S9, S15, unofficial].
2.  **Rotation:** A unit quaternion $\mathbf{q}_k \in \mathbb{R}^4$ ($\|\mathbf{q}_k\|=1$) [S9, unofficial][S14][S16, unofficial].
3.  **Scale:** A vector $\mathbf{s}_k \in \mathbb{R}^3$ representing the scaling factors along the principal axes. These are stored in logarithmic space ($\log \mathbf{s}_k$) to ensure positivity during optimization [S9, unofficial][S14].
4.  **Opacity:** A scalar raw logit $\alpha_k \in \mathbb{R}$. The effective opacity is obtained via a Sigmoid activation: $\hat{\alpha}_k = \sigma(\alpha_k)$ [S9, unofficial][S14].
5.  **Color:** Spherical Harmonics (SH) coefficients $\mathbf{c}_k \in \mathbb{R}^{3 \times (\ell+1)^2}$, where $\ell$ is the degree. Vanilla 3DGS uses degree $\ell=3$, resulting in 16 coefficients per channel (48 floats total) [S9, S16, unofficial].

The covariance matrix $\boldsymbol{\Sigma}_k \in \mathbb{R}^{3 \times 3}$ is derived from rotation and scale:
$$ \boldsymbol{\Sigma}_k = \mathbf{R}_k \mathbf{S}_k \mathbf{S}_k^\top \mathbf{R}_k^\top $$
where $\mathbf{R}_k$ is the rotation matrix constructed from $\mathbf{q}_k$, and $\mathbf{S}_k = \text{diag}(\mathbf{s}_k)$ [S7, S9, S16, unofficial].

**Rendering Pipeline**
1.  **Culling:** Gaussians outside the view frustum or behind the camera are discarded [S9, unofficial].
2.  **Projection:** Each 3D Gaussian is projected to the 2D image plane. The 2D mean $\boldsymbol{\mu}'_k$ is computed via the projection matrix. The 2D covariance $\boldsymbol{\Sigma}'_k$ is approximated using the Jacobian of the projective transformation $J$ and the world-to-camera matrix $W$:
    $$ \boldsymbol{\Sigma}'_k = J W \boldsymbol{\Sigma}_k W^\top J^\top $$
    [S2][S9, S16, unofficial].
3.  **Bounding Box:** A 2D axis-aligned bounding box (AABB) is computed for each Gaussian based on $\boldsymbol{\Sigma}'_k$ (typically expanding the ellipse to cover $3\sigma$) [S7, S9, unofficial].
4.  **Tiling and Sorting:** Gaussians are binned into tiles (e.g., $16 \times 16$ pixels). Within each tile, Gaussians are sorted front-to-back by their depth (center depth) [S7, S9, unofficial].
5.  **Rasterization and Compositing:** For each pixel, the contributing Gaussians are alpha-composited. The color $C(\mathbf{p})$ at pixel $\mathbf{p}$ is:
    $$ C(\mathbf{p}) = \sum_{k \in N} \mathbf{c}_k(\mathbf{d}_k) \, \hat{\alpha}_k \, G_k^{2D}(\mathbf{p}) \prod_{j < k} \left( 1 - \hat{\alpha}_j \, G_j^{2D}(\mathbf{p}) \right) $$
    where $N$ is the set of sorted Gaussians covering the pixel, $\mathbf{d}_k$ is the view direction, and $G_k^{2D}(\mathbf{p})$ is the 2D Gaussian kernel value:
    $$ G_k^{2D}(\mathbf{p}) = \exp\left( -\frac{1}{2} (\mathbf{p} - \boldsymbol{\mu}'_k)^\top (\boldsymbol{\Sigma}'_k)^{-1} (\mathbf{p} - \boldsymbol{\mu}'_k) \right) $$
    [S4][S7, S9, S16, unofficial]. The term $\hat{\alpha}_k G_k^{2D}(\mathbf{p})$ represents the effective alpha of the $k$-th splat at the pixel.
6.  **Loss and Optimization:** The rendered image is compared to the ground truth using a combined L1 and SSIM loss:
    $$ \mathcal{L} = (1 - \lambda) \|\hat{I} - I\|_1 + \lambda (1 - \text{SSIM}(\hat{I}, I)) $$
    with $\lambda = 0.2$ [S9, unofficial]. Gradients are backpropagated through the rasterizer to update $\boldsymbol{\mu}, \mathbf{q}, \mathbf{s}, \alpha, \text{SH}$ using the Adam optimizer [S9, unofficial].

**Adaptive Density Control (ADC)**
To handle local minima and varying detail, 3DGS employs periodic densification and pruning every $K$ iterations (typically every 100 iterations within a window of 500–15,000) [S7, S9, unofficial]:
*   **Gradient Tracking:** The magnitude of the 2D position gradient $\|\nabla_{\boldsymbol{\mu}_{2D}} \mathcal{L}\|$ is tracked for each Gaussian [S9, unofficial].
*   **Clone:** If a Gaussian has small scale and high gradient, it is cloned (duplicated with perturbed position) to fill under-reconstructed areas [S9, unofficial].
*   **Split:** If a Gaussian has large scale and high gradient, it is replaced by two smaller Gaussians (clones with halved scale) to increase resolution [S9, unofficial].
*   **Prune:** Gaussians with opacity below a threshold $\epsilon_\alpha \approx 0.005$ are removed [S9, unofficial].
*   **Opacity Reset:** Every 3,000 iterations, all opacities are reset to a low value (e.g., 0.01) to escape local minima where Gaussians are opaque but incorrect [S9, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_gaussians(gaussians, camera_matrix, image_width, image_height, tile_size=16):
    """
    Renders a scene defined by 3D Gaussians using tile-based rasterization.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - mu: (3,) mean position
            - q: (4,) unit quaternion
            - s: (3,) log-scale
            - alpha_raw: scalar opacity logit
            - sh_coeffs: (48,) spherical harmonic coefficients
            - cov3d: (3,3) 3D covariance matrix (precomputed or computed from q, s)
        camera_matrix: (4,4) world-to-camera projection matrix
        image_width, image_height: int
        tile_size: int
        
    Returns:
        image: (H, W, 3) rendered RGB image
    """
    # 1. Precompute 2D parameters for all Gaussians
    projected_gaussians = []
    for g in gaussians:
        # Compute 3D covariance from quaternion and scale
        R = quaternion_to_rotation_matrix(g.q)
        S = np.diag(np.exp(g.s)) # s is stored in log space
        Sigma_3d = R @ (S @ S.T) @ R.T
        
        # Project to 2D
        # W is the view matrix (top 3 rows of camera_matrix usually, or full 4x4 depending on convention)
        # J is the Jacobian of the perspective projection at mu
        # For simplicity, assume a helper function handles the full projection math
        mu_2d, Sigma_2d, depth, valid = project_gaussian_3d_to_2d(g.mu, Sigma_3d, camera_matrix)
        
        if not valid or depth < 0:
            continue
            
        # Compute effective opacity
        alpha = sigmoid(g.alpha_raw)
        
        # Compute 2D bounding box
        # Expand Sigma_2d to cover 3 sigma (9 in Mahalanobis distance)
        # Approximate AABB from eigenvalues of Sigma_2d
        eigenvalues, _ = np.linalg.eigh(Sigma_2d)
        max_extent = 3.0 * np.sqrt(np.max(eigenvalues))
        
        min_x = int(np.floor((mu_2d[0] - max_extent) / tile_size))
        max_x = int(np.ceil((mu_2d[0] + max_extent) / tile_size))
        min_y = int(np.floor((mu_2d[1] - max_extent) / tile_size))
        max_y = int(np.ceil((mu_2d[1] + max_extent) / tile_size))
        
        projected_gaussians.append({
            'mu_2d': mu_2d,
            'Sigma_2d': Sigma_2d,
            'alpha': alpha,
            'sh_coeffs': g.sh_coeffs,
            'depth': depth,
            'tile_min': (min_x, min_y),
            'tile_max': (max_x, max_y)
        })
        
    # 2. Bin into tiles
    num_tiles_x = (image_width + tile_size - 1) // tile_size
    num_tiles_y = (image_height + tile_size - 1) // tile_size
    tiles = {}
    
    for pg in projected_gaussians:
        for tx in range(pg['tile_min'][0], pg['tile_max'][0] + 1):
            for ty in range(pg['tile_min'][1], pg['tile_max'][1] + 1):
                if 0 <= tx < num_tiles_x and 0 <= ty < num_tiles_y:
                    key = (tx, ty)
                    if key not in tiles:
                        tiles[key] = []
                    tiles[key].append(pg)
                    
    # 3. Render each tile
    image = np.zeros((image_height, image_width, 3))
    
    for (tx, ty), pg_list in tiles.items():
        # Sort by depth (front-to-back)
        pg_list.sort(key=lambda x: x['depth'])
        
        # Define tile pixel range
        x_start = tx * tile_size
        y_start = ty * tile_size
        x_end = min(x_start + tile_size, image_width)
        y_end = min(y_start + tile_size, image_height)
        
        # Rasterize
        for y in range(y_start, y_end):
            for x in range(x_start, x_end):
                # Pixel coordinate relative to image
                # Note: mu_2d is typically in image coordinates
                
                # Accumulate color and transmittance
                T = 1.0
                color = np.zeros(3)
                
                for pg in pg_list:
                    # Compute Mahalanobis distance
                    diff = np.array([x, y]) - pg['mu_2d']
                    inv_Sigma = np.linalg.inv(pg['Sigma_2d'])
                    dist = diff @ inv_Sigma @ diff
                    
                    # Gaussian kernel value
                    G_2d = np.exp(-0.5 * dist)
                    
                    # Effective alpha
                    alpha_eff = pg['alpha'] * G_2d
                    
                    if alpha_eff < 1e-4: # Early out if negligible
                        continue
                        
                    # Get color from SH
                    # View direction d is needed for SH evaluation
                    # Assuming camera center is origin for simplicity or passed in
                    d = get_view_direction(x, y, camera_matrix)
                    c = evaluate_sh(pg['sh_coeffs'], d)
                    
                    # Alpha compositing
                    color += c * alpha_eff * T
                    T *= (1.0 - alpha_eff)
                    
                    if T < 1e-4: # Early out if fully opaque
                        break
                
                image[y, x] = color
                
    return image

def optimize_gaussians(gaussians, training_images, camera_poses, num_iterations=30000):
    """
    Optimizes Gaussian parameters using Adam and Adaptive Density Control.
    """
    optimizer = Adam(gaussians.parameters(), lr=1e-4)
    
    for iter in range(num_iterations):
        # 1. Pick a random training view
        idx = random.choice(len(training_images))
        img = training_images[idx]
        pose = camera_poses[idx]
        
        # 2. Render
        rendered_img = render_gaussians(gaussians, pose, img.width, img.height)
        
        # 3. Compute Loss
        loss = compute_loss(rendered_img, img) # L1 + SSIM
        
        # 4. Backprop
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # 5. Adaptive Density Control (every 100 iters, within window 500-15000)
        if 500 < iter < 15000 and iter % 100 == 0:
            densify_and_prune(gaussians, threshold=2e-4)
            
        # 6. Opacity Reset (every 3000 iters)
        if iter % 3000 == 0:
            reset_opacities(gaussians, new_value=0.01)
            
        # 7. SH Degree Warmup (every 1000 iters)
        if iter % 1000 == 0 and iter > 0:
            increase_sh_degree(gaussians)

def densify_and_prune(gaussians, threshold):
    """
    Clones, splits, and prunes Gaussians based on gradients.
    """
    # Compute 2D position gradients
    grads = compute_gradient_magnitude(gaussians, 'mu_2d')
    
    # Clone small Gaussians with high gradient
    for g in gaussians:
        if g.scale < small_threshold and grads[g.id] > threshold:
            clone_gaussian(g)
            
    # Split large Gaussians with high gradient
    for g in gaussians:
        if g.scale > large_threshold and grads[g.id] > threshold:
            split_gaussian(g)
            
    # Prune low opacity Gaussians
    gaussians = [g for g in gaussians if sigmoid(g.alpha_raw) > 0.005]
```

## Subtopics

### [3D Gaussian Splatting Optimization Parameters](3460-3d-gaussian-splatting-optimization-parameters.md)
Defines the hyperparameters and procedural rules for optimizing a set of 3D Gaussians, specifically introducing adaptive density control (clone, split, prune, opacity reset) and periodic opacity resets to converge to a high-fidelity radiance field.

## Operating characteristics
*   **Time Complexity:** Rendering is $O(K)$ per pixel where $K$ is the number of overlapping Gaussians, but effectively constant due to tiling and culling. Training is dominated by the rasterization and backpropagation, scaling linearly with the number of Gaussians and training views.
*   **Memory Complexity:** $O(K)$, where $K$ is the number of Gaussians. Vanilla 3DGS can require millions of Gaussians (e.g., 1-5 million for synthetic scenes), leading to high VRAM usage during training [S5, S9, unofficial].
*   **Performance:** Achieves real-time rendering speeds (e.g., 68.4 FPS at 1080p on an A100 GPU) [S1]. Training time is significantly faster than NeRF (30-60 minutes vs. hours/days) but slower than Instant-NGP (18.4 minutes to 95% PSNR vs. 2.4 minutes) [S1].
*   **Strengths:** Real-time novel view synthesis, high visual quality (PSNR 33.2 dB on Blender dataset), differentiable end-to-end optimization, simple data structure [S1][S5, S9, unofficial].
*   **Limitations:** Moderate surface accuracy (ellipsoids approximate geometry but do not conform to exact surfaces) [S1]. Popping artifacts can occur due to per-tile depth sorting using center depth rather than per-pixel depth [S9, unofficial]. Heuristic ADC can generate excessive overlapping Gaussians [S7, unofficial]. Not directly compatible with generalizable neural prediction due to non-differentiable pruning/splitting operations [S8].

## Background
3D Gaussian Splatting was introduced as a method to replace implicit neural representations with explicit 3D Gaussians, enabling real-time rendering through GPU rasterization [S1, S2]. It was developed to address the computational bottleneck of ray marching in NeRFs while maintaining high visual fidelity [S5, S9, unofficial]. The method initializes from a sparse Structure-from-Motion point cloud and optimizes parameters end-to-end [S1][S9, unofficial]. It has become a baseline for many subsequent variants in the 3DGS family [S3][S9, unofficial].

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
[S3] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
[S4] DNGaussian: Optimizing Sparse-View 3D Gaussian Radiance Fields with Global-Local Depth Normalization, https://doi.org/10.1109/cvpr52733.2024.01963, fetched 2026-10-26
[S5] 3D Gaussian Splatting | The Future of Real-Time Neural Rendering, https://www.youtube.com/watch?v=iKJd9K7cCxs, fetched 2026-10-27 (unofficial)
[S6] Core Philosophy of 3D Gaussian Splatting | The Idea That Changed 3D Rendering, https://www.youtube.com/watch?v=3Lml90hiQgU, fetched 2026-10-27 (unofficial)
[S7] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
[S8] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S9] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S10] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S11] EndoGaussian: Real-time Gaussian Splatting for Dynamic Endoscopic Scene Reconstruction, https://huggingface.co/papers/2401.12561, fetched 2026-10-30 (unofficial)
[S12] Road to 3D Gaussian Splatting | Nazir Nayal, https://nazirnayal.xyz/blog/2024/road-to-3dgs/, fetched 2026-10-29 (unofficial)
[S13] What is Gaussian Splatting? | PlayCanvas Developer Site, https://developer.playcanvas.com/user-manual/gaussian-splatting/what-is-gaussian-splatting/, fetched 2026-10-29
[S14] Efficient Perspective‐Correct 3D Gaussian Splatting Using Hybrid Transparency, https://doi.org/10.1111/cgf.70014, fetched 2026-10-30
[S15] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S16] Universal Beta Splatting, https://doi.org/10.48550/arXiv.2510.03312, fetched 2026-10-30 (unofficial)
[S17] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
