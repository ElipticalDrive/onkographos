# Mipmap-GS

Also called: Mipmap-Gaussian Splatting

## Overview
Mipmap-GS is a test-time adaptation method that deforms pre-trained 3D Gaussians into scale-adaptive primitives to eliminate aliasing artifacts (zoom-in spikes and zoom-out thickness) caused by fixed Gaussian distributions at varying zoom factors [S1]. Unlike methods that apply static 2D/3D smoothing filters or hierarchical Level-of-Detail selection, Mipmap-GS self-adjusts primitive properties (color, shape, size) and distribution (position) using scale-specific mipmap-like pseudo ground-truth, effectively informing the Gaussians of the new sampling rate [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of single-scale trained base 3D Gaussians $\mathcal{G} = \{G_k\}_{k=1}^K$, where each Gaussian $G_k$ is defined by position, covariance (shape/size), opacity, and spherical harmonic coefficients (color) [S1]. A target zoom factor $N$ [S1].
*   **Output:** A set of optimized, scale-adaptive Gaussians $\mathcal{G}^{opt} = \{G^{opt}_k\}_{k=1}^K$ [S1].

**Data Structures:**
The method operates on the standard 3D Gaussian representation. No new hierarchical data structures (like trees or graphs) are introduced; the adaptation is performed on the flat list of Gaussians [S1].

**Parameters:**
*   $N$: The zoom factor. For zoom-in, $N > 1$ (upsampling by $\times N$). For zoom-out, $N < 1$ (downsampling by $\times 1/N$) [S1].
*   $S$: The number of adaptation iterations. Typically 1,000 (1K) iterations are sufficient for convergence [S1].
*   $\beta$: The learning rate for the optimization step [S1].
*   $r(\cdot)$: The mipmap resizing function. For zoom-in, super-resolution methods like SwinIR are used. For zoom-out, bilinear interpolation is used [S1].

**Algorithm:**
The method constructs a "mipmap-like" pseudo ground-truth (pseudo-GT) at the target observation scale and optimizes the base Gaussians to match this target via a scale-aware guidance loss [S1].

1.  **Base Rendering:** Render the base Gaussians $\mathcal{G}$ to a novel view at the basic scale ($\times 1$) to obtain image $\hat{x}$ [S1].
2.  **Pseudo-GT Construction:** Apply the resizing function $r(\cdot)$ to $\hat{x}$ to generate the scale-specific pseudo-GT $r(\hat{x})$ [S1].
    *   If zooming in ($\times N$), upsample $\hat{x}$ using a super-resolution network (e.g., SwinIR) [S1].
    *   If zooming out ($\times 1/N$), downsample $\hat{x}$ using bilinear interpolation [S1].
3.  **Optimization Loop:** For iteration $i$ from $0$ to $S-1$:
    a. Select a random viewpoint $v_j$ from the training set [S1].
    b. Render the current Gaussians $\mathcal{G}_i$ (initially $\mathcal{G}$) to the basic scale image $\hat{x}_j$ [S1].
    c. Construct the pseudo-GT $r(\hat{x}_j)$ using $r(\cdot)$ and the target zoom factor $N$ [S1].
    d. Render the current Gaussians $\mathcal{G}_i$ to the *new scale* image $x_j$. Note: This rendering must account for the target pixel coverage corresponding to scale $N$ [S1].
    e. Compute the loss $L(x_j, r(\hat{x}_j)) = \|x_j - r(\hat{x}_j)\|_2^2$ [S1].
    f. Update the Gaussians: $\mathcal{G}_{i+1} = \mathcal{G}_i - \beta \nabla L(x_j, r(\hat{x}_j))$ [S1].
    g. If $i \mod 100 == 0$, perform densification or pruning to maintain compactness [S1].
4.  **Result:** Return $\mathcal{G}^{opt} = \mathcal{G}_S$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mipmap_gs_optimize(base_gaussians, viewpoints, zoom_factor, iterations, learning_rate):
    """
    Deforms base 3D Gaussians to be adaptive for a specific zoom scale.
    
    Args:
        base_gaussians: List of Gaussian primitives (position, cov, opacity, SH).
        viewpoints: List of camera poses for training views.
        zoom_factor: Float N. >1 for zoom-in, <1 for zoom-out.
        iterations: Integer S, number of optimization steps (default 1000).
        learning_rate: Float beta.
        
    Returns:
        optimized_gaussians: List of deformed Gaussian primitives.
    """
    current_gaussians = base_gaussians
    
    # Determine resizing function based on zoom direction
    if zoom_factor > 1:
        # Zoom-in: Use super-resolution (e.g., SwinIR)
        resize_func = super_resolution_upsample
    else:
        # Zoom-out: Use bilinear interpolation
        resize_func = bilinear_downsample
        
    for i in range(iterations):
        # 1. Select random viewpoint
        view = random.choice(viewpoints)
        
        # 2. Render base Gaussians at basic scale (x1)
        # This produces the source image for pseudo-GT generation
        image_basic = render_gaussians(current_gaussians, view)
        
        # 3. Construct mipmap-like pseudo-GT at target scale
        # For zoom-in: upsample basic image to xN
        # For zoom-out: downsample basic image to x(1/N)
        pseudo_gt = resize_func(image_basic, zoom_factor)
        
        # 4. Render current Gaussians at the NEW scale
        # The rasterizer must be configured to render at the target resolution/coverage
        # implied by zoom_factor. This effectively simulates the pixel coverage
        # of the target zoom level.
        image_new_scale = render_gaussians_at_scale(current_gaussians, view, zoom_factor)
        
        # 5. Compute L2 loss between rendered image and pseudo-GT
        loss = l2_loss(image_new_scale, pseudo_gt)
        
        # 6. Optimize Gaussians
        # Gradients are computed w.r.t Gaussian parameters
        current_gaussians = optimize_step(current_gaussians, loss, learning_rate)
        
        # 7. Active pruning every 100 iterations
        if i % 100 == 0:
            current_gaussians = densify_and_prune(current_gaussians)
            
    return current_gaussians
```

**Edge Cases and Pitfalls:**
*   **Zoom-in Spikes:** Caused by varying shrinkage bias during accumulation when sampling rates change. Standard 3DGS uses constant dilation which fails here, leading to thin structures or empty areas [S1].
*   **Zoom-out Thickness:** Caused by constant dilation leading to excessively wide coverage and thick structures. Standard 3DGS includes Gaussians in pixels they shouldn't occupy due to inherited dilation [S1].
*   **High-Frequency Loss:** Unlike Mip-Splatting which uses heuristic 2D/3D smoothing filters that suppress high-frequency components, Mipmap-GS preserves finer details by deforming the Gaussians themselves [S1].
*   **Pruning Strategy:** Unlike standard 3DGS which stops density control halfway through training, Mipmap-GS performs active pruning throughout the entire 1K iteration process to ensure a compact representation, especially important for zoom-out where fewer Gaussians are needed [S1].

## Operating characteristics
*   **Time Complexity:** The method requires 1,000 iterations of optimization per target zoom factor. This is significantly faster than training from scratch, requiring only ~3% of the iterations of full training [S1].
*   **Memory Complexity:** Memory usage is comparable to the base 3DGS representation, potentially reduced due to active pruning during optimization [S1].
*   **Performance:** On the NeRF Synthetic dataset, Mipmap-GS outperforms standard 3DGS by an average of 9.25 dB PSNR for zoom-in and 10.40 dB PSNR for zoom-out [S1].
    *   At $\times 1/2$ zoom-out on NeRF Synthetic: SSIM 0.971, PSNR 32.71, LPIPS 0.026 (vs 3DGS: 0.951, 27.14, 0.031) [S1].
    *   At $\times 1/2$ zoom-out on Mip-NeRF 360: SSIM 0.854, PSNR 28.57 (vs 3DGS: 0.774, 26.66) [S1].
*   **Strengths:** Plug-in module applicable to any 3DGS model; solves aliasing essentially by informing the sampling rate rather than filtering; preserves high-frequency details better than filter-based methods [S1].
*   **Limitations:** Requires test-time adaptation for each new zoom factor; relies on super-resolution networks for zoom-in which may introduce artifacts if the base rendering is poor [S1].

## Background
Mipmap-GS was proposed to address the severe aliasing and blurriness in 3D Gaussian Splatting (3DGS) when zooming in or out [S1]. Traditional 3DGS suffers from mismatched pixel coverage and fixed Gaussian distributions at varying zoom factors [S1]. Previous attempts like Mip-Splatting used heuristic 2D/3D smoothing filters, and Octree-GS used hierarchical Level-of-Detail, but these either suppressed high frequencies or lacked accurate pixel coverage selection [S1]. Mipmap-GS introduces a unified optimization method that deforms Gaussians using scale-specific mipmap-like pseudo ground-truth, published in 2025 [S1].

## Sources
[S1] Mipmap-GS: Let Gaussians Deform with Scale-Specific Mipmap for Anti-Aliasing Rendering, https://doi.org/10.1109/3dv66043.2025.00091, fetched 2026-10-27
