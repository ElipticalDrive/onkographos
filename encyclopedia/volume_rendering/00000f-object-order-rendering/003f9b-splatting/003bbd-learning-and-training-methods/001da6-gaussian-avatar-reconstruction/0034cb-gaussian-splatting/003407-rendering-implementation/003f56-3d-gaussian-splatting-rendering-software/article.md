# 3D Gaussian Splatting Rendering (Software)

Also called: 3D Gaussian Splatting (3DGS) Pipeline, 3D Gaussian Splatting Rendering Pipeline, Standard 3D Gaussian Splatting Rendering, Original 3DGS, Kerbl et al. pipeline

## Overview
3D Gaussian Splatting (3DGS) is a novel view synthesis method that represents a scene as a set of unnormalized 3D Gaussian ellipsoids, enabling fast, high-quality real-time rendering by rasterizing projected Gaussians [S5]. It solves the trade-off between the high quality of Neural Radiance Fields (NeRFs) and the speed of traditional point-based rendering by using a differentiable, tile-based software rasterizer [S3, unofficial]. Unlike sibling methods that rely on CPU sorting or hardware ray tracing, this pipeline uses object-level depth sorting followed by tile-based alpha compositing on the GPU, making it the baseline for desktop GPU rendering [S5].

## How it works
The rendering pipeline processes a list of 3D Gaussians to produce a 2D image buffer. It relies on the shared principles of primitive representation, covariance construction, projection, conic representation, filtering, and alpha compositing defined in [Parent](7155-rendering-implementation.md).

### Inputs and Outputs
**Inputs:**
*   **Gaussians:** A list of $N$ primitives. Each primitive $i$ is defined by:
    *   Mean position $\mu \in \mathbb{R}^3$ [S1][S3, unofficial].
    *   Scale $s \in \mathbb{R}^3$ and rotation $q \in \mathbb{R}^4$ (quaternion) [S1][S3, unofficial].
    *   Opacity $\alpha \in \mathbb{R}$ [S1][S3, unofficial].
    *   Spherical Harmonics (SH) coefficients $c_{\ell m}$ for view-dependent color [S1][S3, unofficial]. Degree 3 is standard, requiring $\hat{c} \in \mathbb{R}^{3 \times 16}$ [S3, unofficial].
*   **Camera:** Intrinsics $(f_x, f_y, c_x, c_y)$, extrinsics (view matrix $V$), near plane $z_{near}$, and far plane $z_{far}$ [S3, unofficial][S5].

**Outputs:**
*   **Image:** A 2D buffer of RGB colors and optionally depth/transmittance.

### Data Structures
The Gaussian data is typically stored in a Structure of Arrays (SoA) layout for efficient GPU access [S6, unofficial]:
*   `xyz`: `float32[N, 3]` – Mean positions [S6, unofficial].
*   `rot`: `float32[N, 4]` – Quaternions [S6, unofficial].
*   `scale`: `float32[N, 3]` – Scales [S6, unofficial].
*   `opacity`: `float32[N, 1]` – Opacities [S6, unofficial].
*   `sh`: `float32[N, SH_dim]` – SH coefficients [S6, unofficial].

During rasterization, each Gaussian generates a fragment with:
*   `conic`: `float32[3]` – The inverse 2D covariance $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S2][S6, unofficial].
*   `depth`: `float32` – Camera-space $z$-coordinate $\mu'_z$ [S3, unofficial].
*   `color`: `float32[3]` – Evaluated view-dependent color [S3, unofficial].

### Steps
1.  **Preprocessing (Activation):**
    *   Normalize rotations: $q_i \leftarrow q_i / \|q_i\|$ [S6, unofficial].
    *   Exponentiate scales: $s_i \leftarrow \exp(s_i)$ to ensure positivity [S3, unofficial][S6, unofficial].
    *   Sigmoid opacities: $\alpha_i \leftarrow 1 / (1 + e^{-\alpha_i})$ [S3, unofficial][S6, unofficial].
    *   Concatenate SH coefficients if stored separately [S6, unofficial].

2.  **Covariance Construction:**
    *   Construct rotation matrix $R$ from quaternion $q$ [S1][S3, unofficial].
    *   Construct diagonal scaling matrix $S = \text{diag}(s)$ [S3, unofficial].
    *   Compute 3D covariance: $\Sigma_{3D} = R S S^T R^T$ [S1][S3, unofficial].

3.  **Projection:**
    *   Transform mean to camera space: $\mu' = V \mu$ [S1][S3, unofficial].
    *   Transform covariance to camera space: $\Sigma' = V_{1:3, 1:3} \Sigma_{3D} V_{1:3, 1:3}^T$ [S1][S3, unofficial].
    *   Compute Jacobian $J$ of the projective projection at $\mu'$:
        $$ J = \begin{pmatrix} \frac{f_x}{\mu'_z} & 0 & -\frac{f_x \mu'_x}{\mu'_z^2} \\ 0 & \frac{f_y}{\mu'_z} & -\frac{f_y \mu'_y}{\mu'_z^2} \end{pmatrix} $$
        [S2][S3, unofficial][S5]. Note: $J$ contains zero terms that can be skipped by hardware [S5].
    *   Compute 2D covariance: $\Sigma_{2D} = J \Sigma' J^T$ [S1][S3, unofficial].

4.  **Conic Representation:**
    *   Compute inverse 2D covariance: $\Sigma_{2D}^{-1}$ [S6, unofficial].
    *   Store as conic vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ [S2][S6, unofficial].
    *   *Pitfall:* The code assumes $\Sigma_{2D}$ is non-singular and invertible [S6, unofficial]. If singular, special handling is required [S9].

5.  **Filtering and Sorting:**
    *   **Culling:** Discard Gaussians where $\mu'_z < z_{near}$ [S3, unofficial][S5].
    *   **Sorting:** Sort remaining Gaussians by $\mu'_z$ (depth) to approximate front-to-back order [S3, unofficial][S5]. This avoids prohibitively slow per-pixel sorting [S3, unofficial].

6.  **Rasterization (Tile-based):**
    *   Divide the image into tiles (e.g., $16 \times 16$ pixels) [S5].
    *   For each tile, identify Gaussians whose 2D bounding box overlaps the tile. The bounding box is typically defined by the $3\sigma$ extent of the 2D Gaussian [S6, unofficial].
    *   For each pixel in the tile, iterate through overlapping Gaussians in depth order.

7.  **Alpha Compositing:**
    *   Initialize accumulated color $C_{acc} = 0$ and transmittance $T = 1$ [S5].
    *   For each Gaussian $i$ in depth order:
        *   Evaluate 2D Gaussian at pixel center $\mathbf{x}$:
            $$ \alpha_{pixel} = \alpha_i \cdot \exp\left(-\frac{1}{2} (\mathbf{x} - \mu_{2D})^T \Sigma_{2D}^{-1} (\mathbf{x} - \mu_{2D})\right) $$
            [S3, unofficial][S5]. Truncation is applied at $\approx 3.33\sigma$ [S3, unofficial].
        *   Update color: $C_{acc} \leftarrow C_{acc} + T \cdot \alpha_{pixel} \cdot c_i$ [S3, unofficial][S5].
        *   Update transmittance: $T \leftarrow T \cdot (1 - \alpha_{pixel})$ [S3, unofficial][S5].
        *   **Early Termination:** If $T < \tau$ (threshold), stop processing further Gaussians for this pixel [S5].
    *   Final color: $C_{final} = C_{acc} + T \cdot C_{bg}$ [S3, unofficial][S5].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gaussians(gaussians, camera, image_width, image_height, tile_size=16, tau=1e-4):
    """
    Render 3D Gaussians using the standard 3DGS software pipeline.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - mu: (3,) mean position in world space
            - q: (4,) quaternion rotation
            - s: (3,) scale
            - alpha: (1,) opacity
            - sh: (16, 3) SH coefficients (degree 3)
        camera: Camera object with attributes:
            - view_matrix: (4, 4) world-to-camera matrix
            - fx, fy: focal lengths
            - cx, cy: principal point
            - znear: near plane distance
        image_width, image_height: Output dimensions
        tile_size: Size of rasterization tiles
        tau: Early termination threshold for transmittance
        
    Returns:
        image: (H, W, 3) rendered RGB image
    """
    # 1. Preprocessing and Projection
    projected_gaussians = []
    for g in gaussians:
        # Transform mean to camera space
        mu_homogeneous = np.append(g.mu, 1.0)
        mu_cam = camera.view_matrix @ mu_homogeneous
        mu_prime = mu_cam[:3]
        
        # Check near-plane culling
        if mu_prime[2] < camera.znear:
            continue
            
        # Construct 3D Covariance
        R = quaternion_to_rotation_matrix(g.q)
        S = np.diag(np.exp(g.s)) # Exponentiate scales
        Sigma_3D = R @ S @ S.T @ R.T
        
        # Transform Covariance to Camera Space
        V_3x3 = camera.view_matrix[:3, :3]
        Sigma_prime = V_3x3 @ Sigma_3D @ V_3x3.T
        
        # Compute Jacobian
        zx = mu_prime[0]
        zy = mu_prime[1]
        zz = mu_prime[2]
        J = np.array([
            [camera.fx / zz, 0, -camera.fx * zx / (zz**2)],
            [0, camera.fy / zz, -camera.fy * zy / (zz**2)]
        ])
        
        # Compute 2D Covariance
        Sigma_2D = J @ Sigma_prime @ J.T
        
        # Compute Conic (Inverse 2D Covariance)
        try:
            Sigma_2D_inv = np.linalg.inv(Sigma_2D)
        except np.linalg.LinAlgError:
            continue # Skip singular Gaussians
            
        conic = np.array([Sigma_2D_inv[0,0], Sigma_2D_inv[0,1], Sigma_2D_inv[1,1]])
        
        # Compute 2D Mean
        mu_2d = np.array([
            camera.fx * zx / zz + camera.cx,
            camera.fy * zy / zz + camera.cy
        ])
        
        # Evaluate Color from SH
        # Note: View direction needed for SH evaluation. 
        # Assuming camera center is origin in camera space for direction approximation
        view_dir = -mu_prime / np.linalg.norm(mu_prime)
        color = evaluate_sh(g.sh, view_dir)
        
        # Compute Bounding Box (3-sigma extent)
        # Approximate radius from diagonal elements
        radius_x = 3.0 * np.sqrt(Sigma_2D[0,0])
        radius_y = 3.0 * np.sqrt(Sigma_2D[1,1])
        
        projected_gaussians.append({
            'mu_2d': mu_2d,
            'conic': conic,
            'alpha': g.alpha,
            'color': color,
            'depth': mu_prime[2],
            'radius_x': radius_x,
            'radius_y': radius_y
        })
        
    # 2. Sorting
    projected_gaussians.sort(key=lambda g: g['depth'])
    
    # 3. Rasterization
    image = np.zeros((image_height, image_width, 3))
    
    # Tile-based processing
    for tile_y in range(0, image_height, tile_size):
        for tile_x in range(0, image_width, tile_size):
            tile_h = min(tile_size, image_height - tile_y)
            tile_w = min(tile_size, image_width - tile_x)
            
            # Identify Gaussians overlapping this tile
            tile_gaussians = []
            for g in projected_gaussians:
                # Check bounding box overlap
                if (g['mu_2d'][0] - g['radius_x'] <= tile_x + tile_w and
                    g['mu_2d'][0] + g['radius_x'] >= tile_x and
                    g['mu_2d'][1] - g['radius_y'] <= tile_y + tile_h and
                    g['mu_2d'][1] + g['radius_y'] >= tile_y):
                    tile_gaussians.append(g)
            
            # Process pixels in tile
            for py in range(tile_y, tile_y + tile_h):
                for px in range(tile_x, tile_x + tile_w):
                    pixel_pos = np.array([px, py])
                    C_acc = np.zeros(3)
                    T = 1.0
                    
                    for g in tile_gaussians:
                        # Evaluate Gaussian at pixel
                        diff = pixel_pos - g['mu_2d']
                        exponent = -0.5 * (diff[0]**2 * g['conic'][0] + 
                                           2 * diff[0] * diff[1] * g['conic'][1] + 
                                           diff[1]**2 * g['conic'][2])
                        
                        # Clamp exponent to avoid overflow
                        exponent = max(exponent, -100.0)
                        
                        alpha_pixel = g['alpha'] * np.exp(exponent)
                        
                        C_acc += T * alpha_pixel * g['color']
                        T *= (1.0 - alpha_pixel)
                        
                        if T < tau:
                            break
                            
                    image[py, px] = C_acc + T * camera.background_color
                    
    return image

def evaluate_sh(sh_coeffs, view_dir):
    """
    Evaluate spherical harmonics coefficients for a given view direction.
    Assumes degree 3 (16 coefficients per channel).
    """
    # Standard SH basis evaluation for degree 3
    # This is a simplified placeholder for the actual SH math
    # In practice, this involves evaluating 16 basis functions
    # and dotting with the coefficients.
    # For implementation completeness, we assume a helper exists.
    return sh_coeffs @ sh_basis(view_dir) # [G] Helper for SH basis evaluation

def sh_basis(view_dir):
    """Compute SH basis vector for degree 3 given normalized view direction."""
    x, y, z = view_dir
    # Basis functions for degree 0 to 3
    # c0 = 1/(2*sqrt(pi))
    # c1 = sqrt(3)/(2*sqrt(pi)) * y
    # c2 = sqrt(3)/(2*sqrt(pi)) * z
    # c3 = sqrt(3)/(2*sqrt(pi)) * x
    # ... higher order terms ...
    # This function returns the 16-element vector.
    # [G] Detailed SH basis implementation omitted for brevity, 
    # but required for a complete implementation.
    pass
```

### Parameters
*   **Tile Size:** $16 \times 16$ pixels is standard [S5].
*   **Bounding Box Multiplier:** $3.0$ (representing $3\sigma$ extent) [S6, unofficial].
*   **Near Plane:** $0.01$ [S6, unofficial].
*   **Far Plane:** $100$ [S6, unofficial].
*   **Field of View:** $\pi/2.0$ radians (common default) [S6, unofficial].
*   **Early Termination Threshold ($\tau$):** Not given in the sources.
*   **Training Iterations:** 30,000 iterations per model [S3, unofficial].
*   **Opacity Learning Rate:** 0.025 (halved from 0.05) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Singular Covariance:** The 2D covariance matrix $\Sigma_{2D}$ may become singular (non-invertible), causing `try_inverse()` to fail [S6, unofficial]. Methods like Perspective-Correct 3DGS avoid this by not using matrix inversions [S9].
*   **Projective Approximation:** The Jacobian $J$ is a local affine approximation of the projective transformation. Gaussians are closed under affine transformations but not projective ones, leading to perspective distortion artifacts at close distances [S7].
*   **Sorting Accuracy:** Object-level sorting by $\mu'_z$ is an approximation. It can lead to incorrect blending if Gaussians intersect significantly in depth but are not perfectly sorted per-pixel [S3, unofficial].
*   **Memory Fragmentation:** The adaptive density control during training adds/removes Gaussians, causing memory fragmentation and slowing down optimization in later iterations [S3, unofficial].

## Subtopics

### [3D Gaussian Splatting MVP](3410-3d-gaussian-splatting-mvp.md)
A pure-PyTorch teaching implementation that omits tile-based CUDA rasterization and SH, using readable Python loops.

### [Analytic-Splatting](8084-analytic-splatting.md)
Replaces point-sampling with an analytical integral approximation to solve aliasing artifacts during zooming.

### [3D Gaussian Ray Tracing](8101-3d-gaussian-ray-tracing.md)
Uses BVH-based ray tracing for logarithmic scaling in dense scenes, replacing rasterization.

### [Semantic and Language Embeddings](8130-semantic-and-language-embeddings.md)
Extends 3DGS to store high-dimensional feature vectors for semantic tasks.

### [Human and Articulated Avatars](8209-human-and-articulated-avatars.md)

### [Texture and Appearance Representation](9528-texture-and-appearance-representation.md)

### Perspective-Correct 3D Gaussian Splat Evaluation
A differentiable method for evaluating 3D Gaussians along per-pixel viewing rays that avoids matrix inversion, ensuring numerical stability and eliminating special handling for degenerate splats [S9]. It provides fully perspective-correct rendering, unlike the affine approximation in standard 3DGS which causes distortion at close distances [S9]. It is implemented within a rasterization-based pipeline for real-time performance, unlike ray-tracing frameworks which are slower [S9].

### Fisheye Camera Adaptation (Equidistant Projection)
Adapts the rasterization projection for fisheye cameras using the equidistant projection model [S7]. It computes the 2D coordinate vector $\mathbf{x}_{2D}$ using intrinsic parameters $c_x, c_y, f_x, f_y$ and the pixel coordinates $(u, v)$ via a specific trigonometric formula involving the unit sphere [S7].

### 3D Gaussian Splatting on Mesh Surfaces
Attaches Gaussians onto the surface of a mesh, combining rendered results from a textured human body mesh and Gaussians to reduce the number of Gaussians needed [S1].

### Taming 3DGS
Explores resource-efficiency trade-offs for consumer-grade GPUs using predictable schedules or adaptive importance guidance for pruning/compression [S10, unofficial].

### RadSplat
Leverages robust NeRF initialization to generate a dense point cloud, converted into Gaussians, achieving 900+ FPS rendering with high quality [S10, unofficial].

### 3D Gaussian Splatting with Motion Bases
Positions and rotations change over time, sharing motion coefficients with different motion bases, with regularization for sparsity and local rigidity [S1].

### 3D Gaussian Splatting with Frequency-Domain Regularization
Applies frequency-domain regularization to the rendered 2D image to encourage high-frequency detail recovery [S1].

### 3D Gaussian Splatting with NFD
Utilizes the Generalized Normal Distribution (NFD) to sharpen scene edges [S1].

### Don't Splat your Gaussians
Introduces volumetric ray-traced primitives with closed-form transmittance for efficient media rendering, addressing violations of physical absorption-emission models [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot T)$ where $N$ is the number of Gaussians and $T$ is the number of tiles/pixels, but optimized by tile-based culling and early termination [S5].
*   **Memory Complexity:** $O(N)$ for storing Gaussian attributes [S5]. Unbounded 360-degree scenes can require gigabytes of storage [S5].
*   **Performance:** Targets desktop GPUs for real-time rendering (e.g., 129 FPS Full HD) [S5]. Faces memory/bandwidth limits on mobile/AR-VR edge devices [S5].
*   **Strengths:** Real-time rendering speeds, high-quality novel view synthesis, differentiable optimization [S3, unofficial][S5].
*   **Limitations:** Memory fragmentation during training, perspective distortion artifacts at close distances, poor suitability for resource-constrained devices [S3, unofficial][S5][S7].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. [37] as a novel view synthesis method [S3, unofficial]. It models a scene as a set of unnormalized 3D Gaussian point primitives, each defined by mean, covariance, opacity, and spherical harmonics coefficients [S3, unofficial]. The method uses a tile-based, differentiable software rasterizer for rendering [S3, unofficial]. It has become a baseline for AR/VR and desktop GPU rendering [S5].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] NegGS: Negative Gaussian Splatting, https://doi.org/10.1016/j.ins.2025.121912, fetched 2026-10-26
[S3] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S4] Gaussian-Flow: 4D Reconstruction with Dynamic 3D Gaussian Particle, https://doi.org/10.1109/cvpr52733.2024.01997, fetched 2026-10-26
[S5] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S6] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S7] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
[S8] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
[S9] Efficient Perspective‐Correct 3D Gaussian Splatting Using Hybrid Transparency, https://doi.org/10.1111/cgf.70014, fetched 2026-10-30
[S10] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
