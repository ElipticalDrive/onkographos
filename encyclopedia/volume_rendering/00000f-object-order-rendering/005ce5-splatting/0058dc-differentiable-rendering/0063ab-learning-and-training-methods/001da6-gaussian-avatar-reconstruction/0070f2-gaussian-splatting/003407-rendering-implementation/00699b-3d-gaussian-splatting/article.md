# 3D Gaussian Splatting

Also called: 3D-GS rendering, 3DGS, Bernoulli sampling, R rasterization-based radiance field rendering, Rasterization-based radiance field rendering, Splat Regression Instance

## Overview
3D Gaussian Splatting (3DGS) is a novel view synthesis method that represents a 3D scene as an explicit collection of anisotropic 3D Gaussian primitives, each defined by position, covariance, opacity, and view-dependent color [S1, S11, S15]. Unlike implicit neural radiance fields (NeRFs) that require volumetric ray marching, 3DGS projects these 3D Gaussians into 2D screen-space ellipses and renders them via a tile-based, differentiable rasterization pipeline with alpha compositing [S9, S15, S22]. This approach achieves real-time rendering speeds (100+ FPS) and significantly faster training times compared to NeRFs, while maintaining high photorealistic quality [S8, S27, unofficial]. It sets itself apart from its siblings by using explicit, differentiable Gaussian primitives that allow for fast, tile-based rasterization and direct gradient flow to scene parameters, enabling end-to-end optimization from sparse 2D images [S14, S22].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $N$ 3D Gaussian primitives, each parameterized by:
    *   Mean position $\boldsymbol{\mu} \in \mathbb{R}^3$ [S10, unofficial][S14].
    *   Rotation $q \in \mathbb{R}^4$ (unit quaternion) [S14][S17, unofficial].
    *   Scale $s \in \mathbb{R}^3$ (log-space) [S14][S17, unofficial].
    *   Opacity $\alpha \in \mathbb{R}$ (log-space) [S14][S17, unofficial].
    *   Spherical Harmonics (SH) coefficients $\mathbf{c} \in \mathbb{R}^{3 \times (N_{SH}+1)^2}$ for view-dependent color, typically degree 3 ($N_{SH}=3$, 48 coefficients) [S10, unofficial][S14][S27, unofficial].
*   Camera parameters: Intrinsic matrix $\mathbf{K}$ (focal lengths $f_x, f_y$, principal point $c_x, c_y$) and extrinsic view matrix $\mathbf{V}$ [S10, unofficial][S15].
*   Image dimensions $W \times H$ [S15].

**Outputs:**
*   A rendered RGB image $\mathbf{I} \in \mathbb{R}^{H \times W \times 3}$ [S15][S17, unofficial].

### Data Structures
The scene is stored as a list of Gaussian attributes. A typical layout (e.g., PLY file or GPU buffer) includes:
*   `xyz`: `float32[N, 3]` [S17, unofficial].
*   `rot`: `float32[N, 4]` (quaternions) [S17, unofficial].
*   `scale`: `float32[N, 3]` (log-scale) [S17, unofficial].
*   `opacity`: `float32[N, 1]` (log-opacity) [S17, unofficial].
*   `sh`: `float32[N, SH_dim]` (SH coefficients) [S17, unofficial].

### Parameter Activation
Raw training parameters are transformed into geometric primitives [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md]:
1.  **Rotation**: Normalize the quaternion: $q_{\text{norm}} = q / \|q\|$ [S17, Parent, unofficial].
2.  **Scale**: Exponentiate to ensure positivity: $s_{\text{eff}} = \exp(s)$ [S17, Parent, unofficial].
3.  **Opacity**: Apply sigmoid: $\alpha_{\text{eff}} = \sigma(\alpha) = 1 / (1 + e^{-\alpha})$ [S17, Parent, unofficial].
4.  **Covariance Construction**: Construct the 3D covariance matrix $\boldsymbol{\Sigma}_{3D}$ from the rotation matrix $\mathbf{R}$ (derived from $q_{\text{norm}}$) and scaling matrix $\mathbf{S} = \text{diag}(s_{\text{eff}})$:
    $$ \boldsymbol{\Sigma}_{3D} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T $$
    [S10, Parent, unofficial][S11].

### Camera Space Transformation
1.  Transform the mean position to camera space: $\boldsymbol{\mu}' = \mathbf{V} \boldsymbol{\mu}$ [Parent].
2.  Transform the covariance using the upper-left $3 \times 3$ submatrix of the view matrix $\mathbf{V}$ (denoted $\mathbf{V}_{1:3, 1:3}$):
    $$ \boldsymbol{\Sigma}' = \mathbf{V}_{1:3, 1:3} \boldsymbol{\Sigma}_{3D} \mathbf{V}_{1:3, 1:3}^T $$
    [S10, Parent, unofficial].

### Screen-Space Projection
1.  Compute the Jacobian $\mathbf{J}$ of the projective projection at $\boldsymbol{\mu}'$:
    $$ \mathbf{J} = \begin{pmatrix} \frac{f_x}{\mu'_z} & 0 & -\frac{f_x \mu'_x}{\mu'_z^2} \\ 0 & \frac{f_y}{\mu'_z} & -\frac{f_y \mu'_y}{\mu'_z^2} \end{pmatrix} $$
    [S10, Parent, unofficial][S15].
2.  Compute the 2D screen-space covariance:
    $$ \boldsymbol{\Sigma}_{2D} = \mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^T $$
    [S10, Parent, unofficial][S15].
3.  Compute the 2D mean:
    $$ \boldsymbol{\mu}_{2D} = \begin{pmatrix} \frac{f_x \mu'_x}{\mu'_z} + c_x \\ \frac{f_y \mu'_y}{\mu'_z} + c_y \end{pmatrix} $$
    [S10, Parent, unofficial].
4.  **Truncation**: The 2D Gaussian is truncated at approximately $3.33\sigma$ to determine the bounding box for rasterization [S10, S17, unofficial]. The bounding box size in camera space is derived from the diagonal elements of $\boldsymbol{\Sigma}_{2D}$: $\text{bbox} = 3.0 \times \sqrt{\text{diag}(\boldsymbol{\Sigma}_{2D})}$ [S17, unofficial].

### Rasterization and Alpha Compositing
1.  **Culling**: Discard Gaussians behind the near plane (e.g., $z < 0.01$) [S15][S17, unofficial].
2.  **Sorting**: Sort Gaussians by depth ($\mu'_z$) to approximate front-to-back order. This is typically done using a radix sort per tile [S10, unofficial][S15][S19, unofficial].
3.  **Tile Assignment**: Assign projected 2D Gaussians to image tiles (e.g., $16 \times 16$ pixels) [S1, S15].
4.  **Alpha Blending**: For each pixel, accumulate color $C$ and transmittance $T$ by iterating over depth-sorted Gaussians overlapping the pixel:
    $$ C \leftarrow C + T \cdot \alpha_{\text{pixel}} \cdot \mathbf{c} $$
    $$ T \leftarrow T \cdot (1 - \alpha_{\text{pixel}}) $$
    [S1, S15, Parent].
    Where $\alpha_{\text{pixel}} = \alpha_{\text{eff}} \cdot \exp\left(-\frac{1}{2} (\mathbf{p} - \boldsymbol{\mu}_{2D})^T \boldsymbol{\Sigma}_{2D}^{-1} (\mathbf{p} - \boldsymbol{\mu}_{2D})\right)$ [S1][S10, unofficial].
5.  **Early Termination**: Stop processing Gaussians for a pixel when $T$ drops below a threshold $\tau$ (e.g., 0.05) [S15, Parent].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_3d_gaussians(gaussians, camera, image_width, image_height, tile_size=16):
    """
    Render a scene of 3D Gaussians using tile-based rasterization.
    
    Args:
        gaussians: List of dicts with keys 'xyz', 'rot', 'scale', 'opacity', 'sh'.
                   Shapes: [N, 3], [N, 4], [N, 3], [N, 1], [N, SH_dim].
        camera: Dict with 'K' (3x3), 'V' (4x4), 'near_plane', 'far_plane'.
        image_width, image_height: int.
        tile_size: int.
        
    Returns:
        image: np.ndarray [H, W, 3].
    """
    N = len(gaussians['xyz'])
    
    # 1. Parameter Activation
    # Normalize rotations
    q = gaussians['rot'] / np.linalg.norm(gaussians['rot'], axis=1, keepdims=True)
    # Exponentiate scales
    s_eff = np.exp(gaussians['scale'])
    # Sigmoid opacities
    alpha_eff = 1.0 / (1.0 + np.exp(-gaussians['opacity']))
    
    # 2. Construct 3D Covariance
    # Convert quaternions to rotation matrices R
    R = quaternion_to_rotation_matrix(q) # Helper function
    S_diag = np.diag(s_eff, axis1=-2, axis2=-1) # Shape [N, 3, 3]
    # Sigma_3D = R @ S @ S.T @ R.T
    Sigma_3D = np.einsum('nij,njk,nlk->nil', R, S_diag, S_diag) @ np.einsum('nij,njk->nik', R, S_diag) # Simplified: R S S^T R^T
    # More efficient: Sigma_3D = R @ (S_diag @ S_diag.T) @ R.T
    Sigma_3D = np.einsum('nij,njk,nlk->nil', R, np.einsum('nij,njk->nik', S_diag, S_diag), R)
    
    # 3. Camera Space Transformation
    # Extract upper-left 3x3 of V
    V_3x3 = camera['V'][:3, :3]
    # Transform means
    means_3d_hom = np.concatenate([gaussians['xyz'], np.ones((N, 1))], axis=1) # [N, 4]
    means_cam_hom = (camera['V'] @ means_3d_hom.T).T # [N, 4]
    means_cam = means_cam_hom[:, :3] # [N, 3]
    
    # Transform covariances
    Sigma_cam = np.einsum('ij,njk,lk->nil', V_3x3, Sigma_3D, V_3x3) # [N, 3, 3]
    
    # 4. Screen-Space Projection
    fx, fy = camera['K'][0, 0], camera['K'][1, 1]
    cx, cy = camera['K'][0, 2], camera['K'][1, 2]
    
    # Jacobian J
    # J = [[fx/z, 0, -fx*x/z^2], [0, fy/z, -fy*y/z^2]]
    z = means_cam[:, 2:3] # [N, 1]
    x = means_cam[:, 0:1] # [N, 1]
    y = means_cam[:, 1:2] # [N, 1]
    
    J = np.zeros((N, 2, 3))
    J[:, 0, 0] = fx / z
    J[:, 0, 2] = -fx * x / (z ** 2)
    J[:, 1, 1] = fy / z
    J[:, 1, 2] = -fy * y / (z ** 2)
    
    # Sigma_2D = J @ Sigma_cam @ J.T
    Sigma_2D = np.einsum('nij,njk,nlk->nil', J, Sigma_cam, J) # [N, 2, 2]
    
    # Means_2D
    means_2d = np.zeros((N, 2))
    means_2d[:, 0] = fx * x / z + cx
    means_2d[:, 1] = fy * y / z + cy
    
    # 5. Culling and Bounding Box
    # Near plane culling
    valid = z[:, 0] > camera['near_plane']
    means_2d = means_2d[valid]
    Sigma_2D = Sigma_2D[valid]
    alpha_eff = alpha_eff[valid]
    means_cam = means_cam[valid]
    R_valid = R[valid]
    s_eff_valid = s_eff[valid]
    xyz_valid = gaussians['xyz'][valid]
    sh_valid = gaussians['sh'][valid]
    
    # Compute bounding box size (3 sigma)
    # diag(Sigma_2D) gives variances
    bbox_size = 3.0 * np.sqrt(np.diag(Sigma_2D, axis1=-2, axis2=-1)) # [N, 2]
    
    # 6. Tile Assignment and Sorting
    # Assign to tiles
    tile_ids = (means_2d / tile_size).astype(int)
    # Filter out-of-bounds
    in_bounds = (tile_ids[:, 0] >= 0) & (tile_ids[:, 0] < (image_width + tile_size - 1) // tile_size) & \
                (tile_ids[:, 1] >= 0) & (tile_ids[:, 1] < (image_height + tile_size - 1) // tile_size)
    
    # ... (Implementation of tile-based sorting and rasterization would follow here)
    # For brevity, assuming a standard tile-based rasterizer loop:
    
    image = np.zeros((image_height, image_width, 3))
    
    # Iterate over tiles
    for ty in range(0, image_height, tile_size):
        for tx in range(0, image_width, tile_size):
            # Get Gaussians for this tile
            tile_mask = (tile_ids[:, 0] == tx // tile_size) & (tile_ids[:, 1] == ty // tile_size)
            tile_gaussians_idx = np.where(tile_mask)[0]
            
            if len(tile_gaussians_idx) == 0:
                continue
                
            # Sort by depth
            sorted_idx = tile_gaussians_idx[np.argsort(means_cam[tile_gaussians_idx, 2])]
            
            # Rasterize
            for idx in sorted_idx:
                # Compute alpha at each pixel in the tile's bounding box
                # ... (Pixel-wise alpha computation and compositing)
                pass
                
    return image
```

### Parameters
*   **Tile Size**: Typically $16 \times 16$ pixels [S15][S16, unofficial].
*   **Near Plane**: $0.01$ [S17, S16, unofficial].
*   **Far Plane**: $100$ or $1e10$ [S17, S16, unofficial].
*   **SH Degree**: Typically 3, resulting in 48 coefficients per Gaussian [S10, S27, unofficial].
*   **Early Termination Threshold ($\tau$)**: $0.05$ [S15, Parent].
*   **Bounding Box Multiplier**: $3.0$ (3-sigma extent) [S17, unofficial].
*   **Training Iterations**: 30,000 [S10, unofficial].
*   **Opacity Learning Rate**: $0.025$ (halved from 0.05) [S10, unofficial].

### Edge Cases and Pitfalls
*   **Covariance Inversion**: The 2D covariance matrix $\boldsymbol{\Sigma}_{2D}$ must be invertible. Degenerate Gaussians (e.g., zero scale) can cause singular matrices [S17, unofficial].
*   **Jacobian Rank**: The Jacobian $\mathbf{J}$ has rank 2. The inverse of $\boldsymbol{\Sigma}_{2D}$ is effectively the inverse of the $2 \times 2$ submatrix obtained by skipping the third row/column of the full projected covariance [S23].
*   **Projective Approximation**: The projection is affine, not projective. Gaussians are closed under affine transformations but not projective ones, leading to approximation errors [S23].
*   **Sorting Bottleneck**: Sorting millions of Gaussians is a major bottleneck. Efficient implementations use CUB device radix sort (CUDA) [S19, unofficial].
*   **Initialization**: Random initialization leads to unstable convergence. Sparse SfM point cloud initialization is recommended [S27, unofficial].
*   **Out-of-Domain Views**: Artifacts may appear in unobserved regions [S27, unofficial].

## Subtopics

### [CoR-GS](1486-cor-gs.md)
CoR-GS improves sparse-view reconstruction by defining two independent reconstructions and comparing them to discard inaccurate Gaussians. It differs from feed-forward network approaches by relying on a consistency check between dual optimization paths to prune erroneous primitives. [CoR-GS](1486-cor-gs.md)

### [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)
The 3D Gaussian Splatting Differentiable Rasterizer renders a scene represented by a set of 3D Gaussian primitives into a 2D image by projecting them to screen space, sorting them by depth, and accumulating their contributions via alpha compositing. Unlike volumetric ray marching used in NeRFs, this method operates in object order, rasterizing 2D elliptical kernels (splats) directly onto the image plane. It is distinguished from its siblings by its specific use of a tile-based, front-to-back alpha blending pipeline that is fully differentiable with respect to all Gaussian parameters (position, scale, rotation, opacity) and camera poses, enabling end-to-end gradient-based optimization. [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md)

### [EffGS](3365-effgs.md)
EffGS is an acceleration framework for 3D Gaussian Splatting that improves training and rendering efficiency while maintaining high-fidelity reconstruction in both bounded and large-scale scenes. It sets itself apart from standard 3DGS and other acceleration methods by combining frequency-aware guidance, localized density control, and adaptive primitive scale modulation to prevent quality degradation in complex environments. [EffGS](3365-effgs.md)

### [3D Gaussian Splatting MVP](3410-3d-gaussian-splatting-mvp.md)
3D Gaussian Splatting MVP is a self-contained, pure-PyTorch implementation of the core 3D Gaussian Splatting algorithm, designed as a teaching reference rather than a production renderer. It renders 3D Gaussians onto screen space and rasterizes them using alpha blending to produce smooth, hole-free images from arbitrary viewpoints. It sets itself apart from production implementations by intentionally omitting tile-based CUDA rasterization, spherical harmonics for view-dependent color, and adaptive density control, relying instead on a readable Python loop for the forward pass. [3D Gaussian Splatting MVP](3410-3d-gaussian-splatting-mvp.md)

### [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)
Ghost Ellipsoids for Chunked Training resolves boundary artifacts in chunked volume training by including ellipsoids from neighboring chunks in the loss computation without updating their parameters, providing boundary context without requiring a global volume or merged model. Unlike chunk-based 3DGS methods that resolve boundary inconsistencies by merging chunks into a global coarse model or through post-training stages, this method requires boundary support during training itself without access to a global volume. [Ghost Ellipsoids for Chunked Training](3559-ghost-ellipsoids-for-chunked-training.md)

### [Substitutive LOD with K-wise Moment-Matched Merging](4801-substitutive-lod-with-k-wise-moment-matched-merging.md)
Substitutive LOD with K-wise moment-matched merging is a technique for reducing the number of 3D Gaussians in a scene by merging groups of $K$ splats into a single representative Gaussian, thereby boosting rendering performance and managing memory constraints. It addresses the problem of excessive Gaussian counts and associated aliasing artifacts by replacing small or redundant Gaussians with larger, merged ones. Unlike methods that modify Gaussians during reconstruction or prune based on static importance, this method operates on the final set of Gaussians using cost metrics to guide the merging process, allowing for runtime LOD selection. It sets itself apart from siblings like [CoR-GS](1486-cor-gs.md) and [EffGS](3365-effgs.md) by focusing on substitutive merging via moment matching rather than density control or vector quantization. [Substitutive LOD with K-wise Moment-Matched Merging](4801-substitutive-lod-with-k-wise-moment-matched-merging.md)

### [Dynamic 3D Gaussian Splatting](5703-dynamic-3d-gaussian-splatting.md)
Dynamic 3D Gaussian Splatting extends static 3D Gaussian Splatting to handle time-varying scenes by assigning time-varying position and rotation attributes to each Gaussian primitive. It solves the problem of temporal flicker and storage explosion inherent in per-frame baselines by enforcing physical plausibility through local-rigid and As-Rigid-As-Possible (ARAP) or isometry constraints. Unlike [4D Gaussian Splatting](7992-4d-gaussian-splatting.md) which uses implicit neural fields or planes to predict deformation offsets from a canonical space, this method explicitly tracks the motion of each Gaussian particle, offering a more direct representation of dynamic geometry. [Dynamic 3D Gaussian Splatting](5703-dynamic-3d-gaussian-splatting.md)

### [Epipolar 3D Gaussian Splatting](6070-epipolar-3d-gaussian-splatting.md)
Epipolar 3D Gaussian Splatting is a method that incorporates an epipolar transformer to generalize features between two views, pairing this with a probabilistic prediction of Gaussians aligned along camera-to-pixel rays. [Epipolar 3D Gaussian Splatting](6070-epipolar-3d-gaussian-splatting.md)

### [3D Gaussian Splatting with Transformer-based Parameterization](6072-3d-gaussian-splatting-with-transformer-based-parameterizatio.md)
This method reconstructs a 3D Gaussian Splatting scene from a small set of input images using a Transformer architecture. It tokenizes input views and camera parameters, processes them through a Transformer block, and directly decodes the resulting tokens into 3D Gaussian parameters. [3D Gaussian Splatting with Transformer-based Parameterization](6072-3d-gaussian-splatting-with-transformer-based-parameterizatio.md)

### [SAGA](6154-saga.md)
SAGA is a 2D promptable segmentation method lifted to 3D, leveraging the Segment Anything Model (SAM) for efficient segmentation within 3D Gaussian Splatting (3DGS). It differs from sibling methods like [CoR-GS](1486-cor-gs.md) by relying on SAM's 2D segmentation capabilities to guide 3D Gaussian assignment rather than learned identity encodings or language features. [SAGA](6154-saga.md)

### [Quantile Rendering](7396-quantile-rendering.md)
Quantile Rendering (Q-Render) is a sparse, transmittance-guided sampling strategy designed to efficiently render high-dimensional feature maps from 3D Gaussians. It solves the computational bottleneck of dense volume rendering, where accumulating high-dimensional features along every ray scales poorly with feature dimensionality. Unlike conventional volume rendering, which processes all rasterized Gaussians, Q-Render shifts the Riemann partition from the spatial domain to the transmittance domain, sampling only the most representative "quantile Gaussians" that dominate the ray's transmittance profile. This approach reduces complexity from $O(N C)$ to $O(N + K C)$, enabling real-time rendering of high-dimensional data (e.g., 512-D CLIP embeddings) while maintaining segmentation accuracy. [Quantile Rendering](7396-quantile-rendering.md)

### [4D Gaussian Splatting](7992-4d-gaussian-splatting.md)
4D Gaussian Splatting (4DGS) extends 3D Gaussian Splatting to model dynamic scenes by treating time as a fourth dimension, enabling the reconstruction and real-time rendering of 3D videos from arbitrary viewpoints and timestamps. Unlike static 3DGS which assumes a fixed scene, 4DGS captures spatiotemporal deformations of position, rotation, scale, and appearance, allowing for the generation of novel views at any continuous time $t$. It distinguishes itself from deformation-field methods (e.g., Dynamic3DGS) by maintaining a canonical set of Gaussians transformed by a learned field, and from naive temporal extensions by using coherent 4D covariance structures or explicit spatiotemporal encoders to ensure smooth motion and appearance interpolation. [4D Gaussian Splatting](7992-4d-gaussian-splatting.md)

### [Taming 3DGS](8058-taming-3dgs.md)
Taming 3DGS is a resource-efficiency method for 3D Gaussian Splatting that imposes an explicit cap on the total number of Gaussians to enable high-quality rendering on consumer-grade GPUs. It steers the densification process to spend a fixed budget where it helps most, trading peak quality for faster training and smaller model size. Unlike standard 3DGS which allows unbounded growth, this approach formulates optimization as a mixed-integer problem to maintain a constant Gaussian count. [Taming 3DGS](8058-taming-3dgs.md)

### [Analytic-Splatting](8084-analytic-splatting.md)
Analytic-Splatting is a rendering method that replaces the point-sampling of 2D Gaussians in 3D Gaussian Splatting (3DGS) with an analytical approximation of the integral of the Gaussian signal over the pixel window area to reduce aliasing artifacts during zooming or resolution changes. Unlike [3D Gaussian Splatting](2812-3d-gaussian-splatting.md) which treats pixels as isolated points, or [Mip-Splatting](3410-3d-gaussian-splatting-mvp.md) which uses a hybrid filtering mechanism that suppresses high-frequency components, Analytic-Splatting preserves high-quality details by analytically integrating the signal using a conditioned logistic function approximation of the Gaussian CDF. [Analytic-Splatting](8084-analytic-splatting.md)

### [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)
This family defines the mathematical framework for transforming 3D Gaussian primitives, defined by mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$, into 2D distributions suitable for rasterization [G]. It sets apart from standard perspective projection by supporting optimal tangent-plane projections [3293-optimal-gaussian-splatting.md] and specialized geometries such as cone-beam X-ray scanning [3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md]. [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md)

### [Distributed 3DGS Training](10473-distributed-3dgs-training.md)
Distributed 3DGS Training partitions 3D Gaussian Splatting parameters and datasets across multiple GPUs or HPC nodes to overcome single-GPU memory limits, enabling the training of high-resolution and large-scale scenes (e.g., >100M Gaussians). It sets itself apart from single-GPU baselines by preserving the original 3DGS algorithm and convergence characteristics while using spatial partitioning with ghost cells for boundary consistency and sparse all-to-all communication to distribute the rasterization workload. [Distributed 3DGS Training](10473-distributed-3dgs-training.md)

### [Point-Based Ray Tracing for Visibility](10934-point-based-ray-tracing-for-visibility.md)
Point-Based Ray Tracing for Visibility serves as a sparse visibility oracle that evaluates transmittance along sampled light directions using a bounding volume hierarchy (BVH) of Gaussian proxies, enabling evaluation from any point in space in any direction. Unlike the camera-tied rasterizer, it decouples visibility estimation from screen-space projection, allowing it to act as a drop-in companion to standard 3D Gaussian Splatting (3DGS) by reusing the same EWA-splatting formulation for consistent blending without requiring training mismatch. [Point-Based Ray Tracing for Visibility](10934-point-based-ray-tracing-for-visibility.md)

### [Primitive Representation](11838-primitive-representation.md)
Primitive Representation defines the geometric and probabilistic entities used to model 3D scenes in differentiable splatting methods, serving as the fundamental building blocks for rendering [G]. It solves the problem of representing complex geometry and appearance with explicit, differentiable primitives that can be projected and composited efficiently [G]. What sets it apart from implicit representations (like NeRFs) is the use of explicit, analytically defined density functions (typically Gaussians) that allow for fast, tile-based rasterization and direct gradient flow to scene parameters [G]. It differs from sibling methods like [CoR-GS](1486-cor-gs.md) or [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by focusing on the *definition* of the primitive itself (shape, distribution, dimensionality) rather than the rasterization pipeline or optimization strategy. [Primitive Representation](11838-primitive-representation.md)

### [Shading and Lighting Models](11924-shading-and-lighting-models.md)
This family defines how 3D Gaussian primitives are shaded, blended, and lit to produce the final pixel color, extending the basic rasterization of [3D Gaussian Splatting](2812-3d-gaussian-splatting.md). It solves the limitation of standard splatting, which renders only baked view-dependent colors, by introducing physical material properties, analytic anti-aliasing, and robust transparency handling. It differs from [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md) by focusing on physical material properties and deferred pipelines rather than analytic signal response, and from [Transparency and Blending Models](9274-transparency-and-blending-models.md) by its explicit handling of light transport and occlusion transmittance. [Shading and Lighting Models](11924-shading-and-lighting-models.md)

### [3DGS Radiance Caching](12851-3dgs-radiance-caching.md)
3DGS Radiance Caching is a real-time variance reduction technique for volume path tracing that uses 3D Gaussian Splatting (3DGS) primitives to store and query cached radiance values. It reduces noise by probabilistically accepting cached samples based on throughput luminance and a scaling parameter, optimizing exponentially subsampled Gaussians from noisy path samples to achieve real-time frame rates. Unlike neural radiance caching which requires large memory footprints, this method leverages the compact representation of 3DGS to store attenuated radiance at multiple path lengths, offering a significantly smaller memory footprint. [3DGS Radiance Caching](12851-3dgs-radiance-caching.md)

### [Ray-Splat Intersection Algorithms](13401-ray-splat-intersection-algorithms.md)
Ray-splat intersection algorithms evaluate the contribution of 3D Gaussians by intersecting viewing rays with the Gaussian primitives directly, rather than projecting them onto the image plane [G]. This family provides exact or analytically derived intersections, enabling correct handling of semi-transparency and degenerate splats [G]. Unlike standard 3D Gaussian Splatting (3DGS) which approximates the projection as an affine 2D Gaussian, these methods compute the Gaussian's maximum contribution along each ray [G]. [Ray-Splat Intersection Algorithms](13401-ray-splat-intersection-algorithms.md)

### [Application-Specific Extensions](13465-application-specific-extensions.md)
Application-Specific Extensions modify the standard 3D Gaussian Splatting (3DGS) pipeline to address requirements beyond novel-view synthesis of RGB imagery, such as physical property rendering, explicit geometry extraction, and advanced lighting effects. Unlike standard 3DGS, which optimizes view-dependent spherical harmonic colors for photometric accuracy, these methods replace or augment the attribute representation with scalar densities, feature vectors, or physical parameters, and often introduce post-processing or multi-stage rendering pipelines to support downstream tasks like simulation, medical imaging, or inverse rendering. [Application-Specific Extensions](13465-application-specific-extensions.md)

### [Rasterization Pipeline Optimization](15109-rasterization-pipeline-optimization.md)
Rasterization Pipeline Optimization refers to algorithmic and hardware techniques that accelerate 3D Gaussian Splatting (3DGS) rendering by reducing computational overhead in tile intersection, sorting, and alpha compositing [G]. It solves the bottleneck where standard rasterization overestimates Gaussian extents, processes non-intersecting pixels, and suffers from serial blending latency [Child](3543-speedy-splat-rendering-pipeline.md). What sets this family apart from sibling methods like [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) (which focuses on the differentiable math) or [3D Gaussian Splatting MVP](3410-3d-gaussian-splatting-mvp.md) (which focuses on memory layout) is the explicit restructuring of the rendering pipeline stages—specifically moving culling to parallel preprocess phases, replacing coarse bounding-box tests with precise geometric checks, and co-designing hardware units for projection and sorting [Child](3543-speedy-splat-rendering-pipeline.md)[Child](7329-hardware-accelerated-gaussian-rasterization.md)[Child](8365-adr-gaussian.md). [Rasterization Pipeline Optimization](15109-rasterization-pipeline-optimization.md)

### [Semantic and Structural Extensions](17384-semantic-and-structural-extensions.md)
This family extends 3D Gaussian Splatting by attaching high-dimensional feature vectors to Gaussian primitives or by modeling animatable human avatars from multi-view data. It differs from standard rendering by accumulating these feature vectors via differentiable alpha-blending rather than just color, and from sibling methods by rendering feature maps directly through differentiable alpha-compositing [8130-semantic-and-language-embeddings.md]. For structural extensions, it solves dynamic appearance and geometry reconstruction by maintaining a canonical space representation where Gaussians are deformed via differentiable functions driven by pose parameters before rasterization [8209-human-and-articulated-avatars.md]. [Semantic and Structural Extensions](17384-semantic-and-structural-extensions.md)

### [Optimization and Training](17385-optimization-and-training.md)
Optimization and Training is the iterative process that adjusts the parameters of a set of 3D Gaussian primitives to minimize the difference between rendered images and ground truth views. It solves the problem of reconstructing a photorealistic 3D scene representation from a sparse set of 2D images by differentiating through the rasterization pipeline . Unlike NeRF-based methods which optimize a continuous neural field, 3DGS training optimizes explicit geometric primitives (means, covariances, opacities, and spherical harmonic coefficients), offering significantly faster training and inference speeds while maintaining comparable image quality. The method distinguishes itself through its use of adaptive density control, which dynamically adds or removes Gaussians during training to resolve under-reconstructed or over-reconstructed regions. [Optimization and Training](17385-optimization-and-training.md)

## Operating characteristics
*   **Time Complexity**: Rendering is $O(N)$ per pixel, where $N$ is the number of Gaussians overlapping the pixel. Sorting is $O(N \log N)$ or $O(N)$ with radix sort [S19, unofficial].
*   **Memory Complexity**: $O(N)$, where $N$ is the number of Gaussians. Typical scenes use tens of thousands to millions of Gaussians [S27, unofficial].
*   **Performance**: Achieves up to 100+ FPS in real-time rendering [S8, unofficial].
*   **Strengths**: Real-time rendering, fast training, high-quality novel view synthesis, explicit representation [S1][S8, unofficial][S22].
*   **Limitations**: Scene-specific (requires re-training for new scenes), artifacts in out-of-domain views, memory usage scales with scene complexity [S27, unofficial].
*   **When to Use**: Real-time novel view synthesis, dynamic scene rendering, applications requiring explicit scene representation [S1][S8, unofficial].
*   **When Not to Use**: Scenes with complex, non-Lambertian materials without SH, scenes requiring global illumination beyond baked SH, unknown scenes without re-training [S27, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1][S19, unofficial]. It represents a shift from implicit neural fields (NeRFs) to explicit Gaussian primitives, enabling real-time differentiable rendering [S22]. The method was initially developed for novel view synthesis and 3D reconstruction from multi-view images [S1]. It has since been extended to dynamic scenes (4DGS, Dynamic 3DGS), human avatars (GaussianAvatar, HumanGS), and various other applications [S3, S7, S12]. The core innovation lies in the efficient projection and rasterization of 3D Gaussians, allowing for gradient-based optimization of scene parameters [S1][S19, unofficial].

## Sources
[S1] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
[S2] Splat Regression Models, https://doi.org/10.48550/arxiv.2511.14042, fetched 2026-10-26 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] HUGS: Human Gaussian Splats, https://doi.org/10.1109/cvpr52733.2024.00055, fetched 2026-10-26
[S5] NegGS: Negative Gaussian Splatting, https://doi.org/10.1016/j.ins.2025.121912, fetched 2026-10-26
[S6] SpotLessSplats: Ignoring Distractors in 3D Gaussian Splatting, https://doi.org/10.1145/3727143, fetched 2026-10-27
[S7] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
[S8] 3D Gaussian Splatting | Guest Lecture for "Computer Graphics in the AI Era", https://www.youtube.com/watch?v=MBVmQSA24Yk, fetched 2026-10-27 (unofficial)
[S9] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S10] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S11] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S12] Gaussian-Flow: 4D Reconstruction with Dynamic 3D Gaussian Particle, https://doi.org/10.1109/cvpr52733.2024.01997, fetched 2026-10-26
[S13] Segment Anything in 3D with Radiance Fields, https://doi.org/10.1007/s11263-025-02421-7, fetched 2026-10-26
[S14] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
[S15] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S16] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S17] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S18] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S19] Introduction to 3D Gaussian Splatting - Hugging Face, https://huggingface.co/blog/gaussian-splatting, fetched 2026-10-27 (unofficial)
[S20] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S21] 3D Gaussian Splatting - Paper Explained, Training NeRFStudio, https://learnopencv.com/3d-gaussian-splatting/, fetched 2026-10-28 (unofficial)
[S22] Previous Differentiable Rendering Differentiable rendering is a framework that formulates the image synthesis process as a differentiable function of scene parameters, enabling gradient-based optimiza, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/neural-scene-representations/differentiable-rendering, fetched 2026-10-29
[S23] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
[S24] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
[S25] Monte Carlo Energy Aggregation for Mobile 3D Gaussian Splatting, https://doi.org/10.1007/978-3-032-37167-6_17, fetched 2026-10-29
[S26] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
[S27] 3D Gaussian Splatting 徹底解説, https://zenn.dev/tmb0703/articles/e2e7a48bc5bb2c, fetched 2026-10-30 (unofficial)
[S28] “Real-time Photorealistic Dynamic Scene Representation and Rendering with 4D Gaussian Splatting”, https://radiancefields.com/papers/real-time-photorealistic-dynamic-scene-representation-and-rendering-with-4d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S29] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S30] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S31] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S32] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
