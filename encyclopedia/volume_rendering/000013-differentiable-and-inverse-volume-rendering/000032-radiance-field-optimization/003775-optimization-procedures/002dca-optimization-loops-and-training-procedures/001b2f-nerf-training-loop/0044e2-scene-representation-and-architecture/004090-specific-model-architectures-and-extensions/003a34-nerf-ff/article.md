# NeRF-FF

Also called: Focus Frustums, NeRF-FF + iNGP, NeRF-FF + nerfacto, NeRF-FF + splatfacto, Neural Radiance Fields with Focus Frustums

## Overview
NeRF-FF is a plug-in strategy that mitigates the degradation of novel view synthesis caused by out-of-focus (defocus blurred) input images by estimating binary masks that identify in-focus regions [S1]. It enables subsequent NeRF models to omit out-of-focus image regions during training, allowing the network to learn an in-focus representation of the scene [S1]. Unlike methods that use dynamic ray generation and blur kernels (e.g., PDRF, Deblur-NeRF), NeRF-FF relies on static ray assumptions, making it compatible with runtime-optimized variants like Instant-NGP, nerfacto, and splatfacto [S1].

## How it works
NeRF-FF operates as a preprocessing pipeline that generates per-image binary masks. These masks are applied during the training of a subsequent NeRF model to restrict optimization to in-focus pixels [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Set of blurry input images $I_i \in [0, 255]^{3 \times W \times H}$ for $i = 1 \dots N$ [S1].
    *   Hyperparameters: $\tau$ (Laplacian threshold), $\epsilon$ (refinement coverage fraction) [S1].
*   **Outputs:**
    *   Binary masks $Mask_i(x,y)$ for each input image $I_i$ [S1].
    *   A trained NeRF model (e.g., Instant-NGP) optimized only on pixels where $Mask_i(x,y) = 1$ [S1].

### Data Structures
*   **Depth Maps:** $D_i \in \mathbb{R}^{W \times H}$, representing the per-view depth estimated by the preliminary NeRF [S1].
*   **Indicator Pixels:** $P_i$, a set of 2D coordinates $(x,y)$ indicating in-focus regions [S1].
*   **Focus Frustum (FF):** A volume defined by the camera view frustum clipped by a near plane depth $n_i$ and a far plane depth $f_i$ [S1].
*   **Refined Focus Frustum (RFF):** A volume defined by updated near plane $\hat{n}_i$ and far plane $\hat{f}_i$ [S1].
*   **Scene Points:** $S$, a set of high-density 3D points sampled from the scene geometry [S1].

### Parameters
*   $\tau$: Threshold for the discrete Laplace operator. Typical value: 80 [S1].
*   $\epsilon$: Hyperparameter controlling the trade-off between scene coverage and reconstruction quality. It defines the minimum fraction of refined frustums that must encompass each sampled point. Typical value: 0.4 [S1]. Range: $[0, 1.0]$. $\epsilon=0$ omits refinement; $\epsilon=1$ implies no NeRF-FF application [S1].

### Algorithm Steps

1.  **Preliminary Reconstruction:** Train a preliminary NeRF model (e.g., Instant-NGP) on the blurry input images $I_i$ to obtain dense depth maps $D_i$ [S1].
    *   *Note:* The preliminary model is trained for 2,000 iterations in the reference implementation [S1].
    *   *Pitfall:* If inputs contain fully blurred images, this step may produce floater artifacts (high optical density with no ground truth counterpart), leading to inferior depth maps and poor mask quality [S1].

2.  **Identify Indicator Pixels:** Compute the discrete Laplace operator $\square$ on the grayscale version of each input image $I_i$. Identify indicator pixels $P_i$ where the Laplace value exceeds $\tau$:
    $$ P_i = \{ (x, y) \mid \square(I_i)(x, y) > \tau \} $$
    [S1]

3.  **Initial Focus Frustum Estimation:** For each image $i$, sample the depth map $D_i$ at the indicator pixels $P_i$. Perform statistical analysis on these depths to estimate the initial near plane $n_i$ and far plane $f_i$, defining the initial Focus Frustum $FF_i$ [S1].

4.  **Focus Frustum Refinement:** Iteratively expand the near and far planes of $FF_i$ to obtain refined planes $\hat{n}_i$ and $\hat{f}_i$. The expansion minimizes the accumulated depth change $R$ while ensuring that every point $p \in S$ is encompassed by at least a fraction $\epsilon$ of the refined frustums [S1].
    *   *Constraint:* Only frustums whose view frustum encompasses point $p$ are considered for the calculation of $R$ for that point [S1].
    *   *Goal:* Ensure adequate scene geometry coverage while minimizing the inclusion of out-of-focus volumes [S1].

5.  **Mask Generation:** Reproject the dense scene points encompassed by the refined Focus Frustums $RFF_i$ into the image space to generate binary masks $Mask_i$ [S1]. For a pixel $(x,y)$ in image $i$:
    $$ Mask_i(x, y) = \begin{cases} 1, & \text{if } D_i(x, y) \in [\hat{n}_i, \hat{f}_i] \\ 0, & \text{otherwise} \end{cases} $$
    [S1]
    *   *Note:* Since all pixels in the depth map are visible, other boundaries of the frustum (horizontal/vertical) do not need explicit consideration for the mask generation logic described [S1].

6.  **Final Training:** Train the subsequent NeRF model using the input images $I_i$ and the generated masks. Only pixels $I_i(x,y)$ where $Mask_i(x,y) = 1$ are used for the loss calculation [S1].
    *   *Note:* In the reference implementation, the subsequent Instant-NGP is trained for 4,000 iterations [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerff_pipeline(input_images, tau=80, epsilon=0.4):
    """
    input_images: List of images I_i with shape (H, W, 3), values [0, 255]
    tau: Threshold for Laplacian operator
    epsilon: Fraction of frustums covering each scene point
    """
    n_images = len(input_images)
    masks = []
    
    # Step 1: Preliminary Reconstruction
    # Train preliminary NeRF (e.g., Instant-NGP) on input_images
    # Returns depth maps D_i for each image
    depth_maps = train_preliminary_nerf(input_images)
    
    # Step 2: Identify Indicator Pixels
    indicator_pixels = []
    for i in range(n_images):
        I_i = input_images[i]
        # Convert to grayscale
        I_gray = rgb_to_grayscale(I_i)
        # Apply discrete Laplace operator
        laplacian_map = apply_discrete_laplacian(I_gray)
        # Threshold to find sharp edges
        P_i = get_pixels_above_threshold(laplacian_map, tau)
        indicator_pixels.append(P_i)
        
    # Step 3: Initial Focus Frustum Estimation
    # Estimate initial near (n_i) and far (f_i) planes from depth at P_i
    initial_frustums = []
    for i in range(n_images):
        D_i = depth_maps[i]
        P_i = indicator_pixels[i]
        # Sample depths at indicator pixels
        depths_at_indicators = [D_i[p.y, p.x] for p in P_i]
        # Statistical analysis to determine n_i and f_i
        # (e.g., min/max or percentile-based estimation)
        n_i, f_i = estimate_depth_range(depths_at_indicators)
        initial_frustums.append({'n': n_i, 'f': f_i})
        
    # Step 4: Focus Frustum Refinement
    # Sample high-density scene points S from the preliminary NeRF
    scene_points = sample_high_density_points(preliminary_nerf)
    
    refined_frustums = []
    for i in range(n_images):
        # Initialize refined planes with initial estimates
        n_hat = initial_frustums[i]['n']
        f_hat = initial_frustums[i]['f']
        
        # Iteratively expand planes to satisfy epsilon constraint
        # while minimizing accumulated depth change R
        # This involves checking if each point p in scene_points
        # is covered by at least epsilon fraction of frustums
        # and adjusting n_hat, f_hat to minimize R
        
        # Note: The exact optimization function for R is not fully detailed
        # in the source, but it minimizes depth change subject to coverage.
        n_hat, f_hat = refine_frustum_planes(
            initial_frustums[i], 
            scene_points, 
            epsilon, 
            depth_maps
        )
        
        refined_frustums.append({'n': n_hat, 'f': f_hat})
        
    # Step 5: Mask Generation
    for i in range(n_images):
        D_i = depth_maps[i]
        n_hat = refined_frustums[i]['n']
        f_hat = refined_frustums[i]['f']
        
        # Generate binary mask
        # Mask_i(x,y) = 1 if depth is within [n_hat, f_hat]
        mask = np.zeros((D_i.shape[0], D_i.shape[1]), dtype=bool)
        mask[(D_i >= n_hat) & (D_i <= f_hat)] = True
        masks.append(mask)
        
    return masks

def train_final_nerf(input_images, masks, model_type='Instant-NGP'):
    """
    Train the final NeRF model using only pixels where mask is True.
    """
    # Filter training data based on masks
    training_data = []
    for i in range(len(input_images)):
        I_i = input_images[i]
        mask = masks[i]
        # Extract valid pixels
        valid_indices = np.where(mask)
        valid_colors = I_i[valid_indices]
        # Corresponding rays/positions would be derived from camera poses
        # and pixel coordinates
        training_data.append((valid_colors, valid_indices))
        
    # Train model on training_data
    final_model = train_nerf_model(training_data, model_type)
    return final_model
```

### Edge Cases and Pitfalls
*   **Fully Blurred Images:** NeRF-FF is not suited for datasets containing fully blurred images. These cause floater artifacts in the preliminary reconstruction, leading to erroneous depth estimates and inferior masks [S1].
*   **Low $\tau$:** Low threshold values cause bad performance because weak signals (texture in out-of-focus regions) are considered in-focus indicators [S1].
*   **High $\tau$:** High threshold values cause slowly decreasing performance as sharp edges in in-focus regions are missed. However, the decline is small because the initial frustums are refined [S1].
*   **Dense Reconstruction Quality:** Mismatches between in-focus indicators and the spatial position of scene geometry cause erroneous depth estimates, negatively influencing initial Focus Frustum estimates [S1].
*   **Floater Artifacts:** Even with masking, the subsequent reconstruction may suffer from floater artifacts (high optical density with no ground truth counterpart), which reduce image quality depending on the camera position [S1].

## Operating characteristics
*   **Time Complexity:** NeRF-FF reduces training time to under 1 minute on end-consumer hardware (RTX 3090) for the combined NeRF-FF execution and Instant-NGP training [S1]. This constitutes an acceleration of two orders of magnitude compared to state-of-the-art methods like PDRF or Deblur-NeRF [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:**
    *   On the real-world defocus dataset by Ma et al., NeRF-FF + iNGP achieves PSNR on par with SOTA methods (Deblur-NeRF, DP-NeRF, PDRF) and consistently outperforms them in SSIM [S1].
    *   It trails reference deblurring strategies slightly in LPIPS [S1].
    *   On the Wu et al. dataset, NeRF-FF + iNGP produces better quality images than DoF-NeRF [S1].
    *   On synthetic datasets with fully blurred images, NeRF-FF underperforms SOTA due to low-quality dense reconstruction [S1].
*   **Compatibility:** Compatible with Instant-NGP, nerfacto, and splatfacto (3D Gaussian Splatting) [S1]. Applying NeRF-FF masks to nerfacto and splatfacto improves PSNR, SSIM, and LPIPS with negligible impact on training duration [S1].
*   **Strengths:**
    *   Significant training time reduction (two orders of magnitude) [S1].
    *   Compatibility with static ray-based optimized NeRF variants [S1].
    *   Plug-in nature allows easy integration with various NeRF architectures [S1].
*   **Limitations:**
    *   Sensitive to the quality of the preliminary NeRF's dense reconstruction [S1].
    *   Not suitable for fully blurred images [S1].
    *   May suffer from floater artifacts in the final reconstruction [S1].

## Background
NeRF-FF was proposed by T. Wirth et al. in the paper "NeRF-FF: a plug-in method to mitigate defocus blur for runtime optimized neural radiance fields" [S1]. The method addresses the issue that NeRF reconstruction quality suffers significantly from out-of-focus regions in input images [S1]. Existing methods often use dynamic ray generation, which is incompatible with the static ray assumptions of performance-optimized NeRFs like Instant-NGP [S1]. NeRF-FF was introduced to bridge this gap, enabling the use of accelerated NeRF variants on defocus-blurred data [S1].

## Sources
[S1] NeRF-FF: a plug-in method to mitigate defocus blur for runtime optimized neural radiance fields, https://doi.org/10.1007/s00371-024-03507-y, fetched 2026-10-26
