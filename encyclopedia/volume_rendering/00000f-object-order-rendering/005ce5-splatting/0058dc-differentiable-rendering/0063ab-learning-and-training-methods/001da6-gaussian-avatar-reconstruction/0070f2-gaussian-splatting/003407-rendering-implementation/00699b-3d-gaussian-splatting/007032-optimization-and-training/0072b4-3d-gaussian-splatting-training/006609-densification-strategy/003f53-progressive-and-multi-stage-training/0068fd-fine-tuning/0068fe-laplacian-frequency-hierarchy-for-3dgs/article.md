# Laplacian Frequency Hierarchy for 3DGS

Also called: Frequency-staged Training Pipeline, Frequency-staged Training with Archiving, Laplacian Frequency Hierarchies Training Procedure, Laplacian-GS inheritance strategy

## Overview
This method reduces 3D Gaussian Splatting (3DGS) training time and memory usage by factorizing the scene representation into a hierarchy of frequency-specific Gaussian fields aligned with a Laplacian pyramid of training images [S1]. It optimizes these fields sequentially from coarse to fine, archiving completed fields to free GPU memory, and composes the final image in the image domain [S1]. Unlike simultaneous multi-field optimization, this approach reduces the number of active primitives on the GPU at any given stage, offering a speedup with minor quality trade-offs [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training views $\{v\}$ with ground truth images $\{I^v\}$ at full resolution; a camera model for each view.
*   **Outputs:** A set of Gaussian fields $\{G^0, \dots, G^{L-1}\}$ representing the scene across frequency bands.

**Data Structures:**
*   **Gaussian Field $G^\ell$:** A set of $N_\ell$ anisotropic Gaussian primitives. Each primitive $i$ is parameterized by:
    *   Mean position $\mu_i \in \mathbb{R}^3$
    *   Opacity $\sigma_i \in (0, 1)$
    *   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, typically decomposed as $\Sigma_i = R_i S_i S_i^\top R_i^\top$ where $R_i$ is a rotation quaternion and $S_i$ is a diagonal scale matrix [S1].
    *   Spherical Harmonics (SH) coefficients for view-dependent color.
*   **Laplacian Pyramid:** A hierarchy of images $\{I^{L-1}, \dots, I^0\}$ where $I^0$ is the full-resolution ground truth and $I^{L-1}$ is the coarsest low-frequency approximation [S1].
*   **Cache:** A storage for accumulated rendered predictions $\{\hat{I}^v_\ell\}$ used to reconstruct finer levels [S1].

**Preprocessing: Pyramid Construction**
1.  For each training view $v$, construct the resolution pyramid:
    $$I^{\ell+1} = \text{Down}(I^\ell)$$
    where $\text{Down}(\cdot)$ is Gaussian filtering followed by subsampling [S1].
2.  Compute Laplacian residual targets for levels $\ell \in \{0, \dots, L-2\}$:
    $$R^\ell = I^\ell - \text{Up}(I^{\ell+1})$$
    where $\text{Up}(\cdot)$ is the paired upsampling operator [S1].
    Note: $I^{L-1}$ is the low-frequency target for the coarsest field. Residuals $R^\ell$ are signed and lie in $[-1, 1]$, while base images lie in $[0, 1]$ [S1].

**Rendering Functions:**
*   **Render_base:** Standard 3DGS rasterization with post-processing to map SH colors to $[0, 1]$:
    $$c_i(d) = \text{clamp}(c_i(d) + 0.5, 0, 1)$$
    Used for the coarsest level $G^{L-1}$ [S1].
*   **Render_res:** Modified rasterization that removes the shift and clamp, allowing signed outputs in $[-1, 1]$ to match residual targets [S1].

**Algorithm: Frequency-staged Training with Archiving**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def laplacian_frequency_hierarchy_training(training_views, camera_models, L=2, iterations_per_level=[10000, 20000], inheritance_ratio=1.0):
    """
    L: Number of pyramid levels. Default 2 (levels 0 and 1).
    iterations_per_level: List of iterations for each level, from coarsest (L-1) to finest (0).
    inheritance_ratio: Fraction of Gaussians inherited from previous stage.
    """
    
    # 1. Precompute Laplacian Pyramids for all training views
    # I_pyramid[v][l] stores the image at level l for view v
    I_pyramid = {}
    for v in training_views:
        I_full = get_ground_truth_image(v)
        I_pyramid[v] = []
        current_img = I_full
        for l in range(L):
            I_pyramid[v].append(current_img)
            if l < L - 1:
                current_img = gaussian_downsample(current_img)
    
    # 2. Initialize cache for accumulated predictions
    # Cache stores the reconstructed image at level l for view v
    prediction_cache = {} 
    
    # 3. Iterate from coarsest level (L-1) to finest (0)
    gaussian_fields = {}
    
    for l in range(L - 1, -1, -1):
        # Initialize Gaussian Field G^l
        if l == L - 1:
            # Coarsest level: Initialize from COLMAP point cloud
            G_l = initialize_from_colmap(training_views)
        else:
            # Finer level: Inherit from previous stage G^{l+1}
            G_prev = gaussian_fields[l + 1]
            G_l = inherit_gaussians(G_prev, inheritance_ratio)
        
        # 4. Training Loop for Level l
        num_iters = iterations_per_level[L - 1 - l] # Map index to correct iteration count
        optimizer = Adam(G_l.parameters())
        
        for iteration in range(num_iters):
            # Sample a training view
            v = sample_view(training_views)
            
            # Render current field
            if l == L - 1:
                # Coarsest level: Render base image
                I_hat_l = render_base(G_l, v)
            else:
                # Finer level: Render residual
                # Load cached prediction from previous (coarser) level
                I_hat_l_plus_1 = prediction_cache[(v, l + 1)]
                I_upsampled = upsample(I_hat_l_plus_1)
                
                # Render residual field
                R_hat_l = render_res(G_l, v)
                
                # Reconstruct full image at this level for loss computation
                I_hat_l = I_upsampled + R_hat_l
            
            # Compute Photometric Loss
            # Supervise using the ground truth at the current pyramid level I^v_l
            # This anchors the stage to the correct resolution and reduces error accumulation [S1]
            target_I = I_pyramid[v][l]
            loss = photometric_loss(I_hat_l, target_I) # Weighted L1 + SSIM
            
            # Backpropagate and update
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            
            # Apply densification and pruning specific to 3DGS
            apply_densification_and_pruning(G_l, iteration)
        
        # 5. Archive and Cache
        # Cache the accumulated prediction for the next finer stage
        # Note: In practice, only cache I_hat_l in the dataloader; 
        # the Gaussian field G_l is archived (moved off GPU) [S1]
        prediction_cache[(v, l)] = I_hat_l # Simplified: cache for all views or lazy load
        
        # Save and archive G_l (remove from GPU memory)
        gaussian_fields[l] = G_l
        archive_field(G_l)
        
    return gaussian_fields

def inherit_gaussians(G_prev, rho):
    """
    Inherits Gaussians from the previous stage.
    Strategy: Copy only 3D positions (xyz). Reinitialize scale, opacity, rotation, colors.
    If rho < 1.0, randomly select a subset of Gaussians.
    """
    G_new = EmptyGaussianField()
    
    # Determine number of Gaussians to inherit
    N_prev = len(G_prev)
    N_new = int(N_prev * rho)
    
    # Randomly select indices if rho < 1.0, otherwise all
    if rho < 1.0:
        indices = random.sample(range(N_prev), N_new)
    else:
        indices = range(N_prev)
        
    for i in indices:
        g = G_prev[i]
        # Copy only position
        new_g = Gaussian()
        new_g.mu = g.mu
        
        # Reinitialize other attributes
        new_g.sigma = random_uniform(0, 1) # Opacity
        new_g.S = random_scale() # Scale
        new_g.R = random_rotation() # Rotation
        new_g.SH = random_sh_coefficients() # Colors
        
        G_new.add(new_g)
        
    return G_new

def gaussian_downsample(img):
    # Gaussian filtering followed by subsampling (e.g., 2x)
    filtered = gaussian_filter(img, sigma=1.0)
    return filtered[::2, ::2, :]

def upsample(img):
    # Paired upsampling operator (e.g., nearest neighbor or bilinear)
    return resize(img, scale_factor=2.0)
```

**Parameters:**
*   **$L$ (Hierarchy Depth):** Number of pyramid levels. Default is 2 (levels 0 and 1) [S1]. Deeper hierarchies make finer residuals harder to represent efficiently, potentially reducing gains [S1].
*   **$\rho$ (Inheritance Ratio):** Fraction of Gaussians inherited from the previous stage. Default is 1.0 [S1]. Values $< 1.0$ accelerate training but cause a clear drop in reconstruction quality [S1].
*   **Iterations:** Default total 30K iterations split as 10K for the coarsest level and 20K for the finest level in a two-level setup [S1].
*   **Optimizer:** Adam [S1].

**Edge Cases and Pitfalls:**
*   **Error Accumulation:** Training each field to directly match its Laplacian residual target ($R^\ell$) can lead to accumulated approximation errors and degraded quality. The method mitigates this by supervising each stage using the reconstructed image $\hat{I}^v_\ell$ against the ground truth at that resolution level $I^v_\ell$ [S1].
*   **Inheritance Ratio:** Reducing $\rho$ below 1.0 speeds up training but significantly reduces fidelity because fewer primitives remain to represent fine details [S1].
*   **Hierarchy Depth:** Using too many levels (deep hierarchies) can degrade quality because high-frequency residuals become difficult for 3DGS to represent efficiently [S1].

## Subtopics

### Multi-stage and Progressive Training
A broader category of methods that optimize scene representations in stages, often coarse-to-fine. Laplacian Frequency Hierarchy is a specific implementation using frequency decomposition and archiving. [Multi-stage and Progressive Training](18485-multi-stage-and-progressive-training.md)

### Pruning and Sparsity Control
Methods that reduce the number of Gaussians to save memory and speed up training. Laplacian Frequency Hierarchy uses archiving (moving fields off GPU) and optional inheritance ratio reduction to manage active primitives. [Pruning and Sparsity Control](14431-pruning-and-sparsity-control.md)

### FastGS
A backbone framework used to implement Laplacian Frequency Hierarchy. FastGS provides optimized rendering and training routines. [FastGS](11745-splatformer-training-objective-refinement.md) (Note: FastGS is a sibling in the tree, but Laplacian Frequency Hierarchy is implemented *on top* of it. The link points to the sibling article for context on the backbone).

## Operating characteristics
*   **Time Complexity:** Reduces training time by optimizing fewer Gaussians per stage and reducing per-iteration cost at coarser levels. On Mip-NeRF 360 1K, it achieves a 1.73x speedup (8.88 to 5.13 minutes) with Taming-3DGS and 1.22x (4.82 to 3.96 minutes) with FastGS [S1].
*   **Memory Complexity:** Significantly reduces peak VRAM usage by archiving completed Gaussian fields off the GPU. This allows training higher-resolution scenes (2K/4K) that might otherwise OOM [S1].
*   **Performance:** Slight degradation in reconstruction quality (e.g., 0.34 dB PSNR drop on Mip-NeRF 360 1K) is attributed to approximation errors from hierarchical rendering and composition [S1].
*   **Strengths:** Efficient training for high-resolution scenes; reduced memory footprint; leverages standard 3DGS densification/pruning.
*   **Limitations:** Slight quality drop; deeper hierarchies may degrade quality; requires careful tuning of inheritance ratio $\rho$ if speed is prioritized over quality.
*   **When to Use:** When training time or VRAM is a bottleneck for high-resolution 3DGS scenes.
*   **When Not to Use:** When maximum reconstruction fidelity is required and training time is not a constraint.

## Background
The method was introduced in "Laplacian Frequency Hierarchies for Efficient 3D Gaussian Splatting Training" (2024/2025) [S1]. It builds on the observation that low-frequency structures in 3DGS converge earlier than high-frequency details, and that standard 3DGS rasterizers assume non-negative colors, which is incompatible with signed residual learning. The Laplacian pyramid (Burt & Adelson, 1983) provides a framework for decomposing images into frequency bands, allowing separate optimization of coarse and fine details [S1].

## Sources
[S1] Laplacian Frequency Hierarchies for Efficient 3D Gaussian Splatting Training, https://doi.org/10.48550/arxiv.2609.03334, fetched 2026-10-26
