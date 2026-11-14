# NeRF-FF

Also called: Focus Frustums, NeRF-FF + iNGP, NeRF-FF + nerfacto, NeRF-FF + splatfacto, Neural Radiance Fields with Focus Frustums

## Overview
NeRF-FF is a plug-in strategy that mitigates the degradation of Neural Radiance Field (NeRF) reconstruction quality caused by out-of-focus (defocus blurred) regions in input images [S1]. It solves the problem of training NeRFs on partially blurred data by estimating binary image masks that identify in-focus regions, allowing the subsequent NeRF model to omit out-of-focus pixels during training [S1]. Unlike methods that use dynamic ray generation or blur kernels (e.g., PDRF, Deblur-NeRF), NeRF-FF relies on static ray assumptions, making it compatible with runtime-optimized NeRF variants like Instant-NGP, nerfacto, and splatfacto [S1].

## How it works
NeRF-FF operates as a preprocessing pipeline that generates per-image binary masks [S1]. These masks are applied during the training of a subsequent NeRF model (e.g., Instant-NGP) to restrict optimization to in-focus pixels [S1].

### Inputs and Outputs
*   **Inputs:** A set of $N$ partially defocus-blurred input images $\{I_i\}_{i=1}^N$, where $I_i \in [0, 255]^{3 \times W \times H}$ represents the color information for image $i$ with width $W$ and height $H$ [S1].
*   **Outputs:** A set of binary masks $\{Mask_i\}_{i=1}^N$, where $Mask_i(x, y) \in \{0, 1\}$ indicates whether pixel $(x, y)$ in image $i$ is in-focus [S1].

### Data Structures
1.  **Preliminary NeRF Model:** A trained NeRF instance (e.g., Instant-NGP) used to estimate scene geometry [S1].
2.  **Depth Maps:** For each image $i$, a dense depth map $D_i \in \mathbb{R}^{W \times H}$ is derived from the rays deployed during the volume rendering process of the preliminary NeRF inference [S1]. $D_i(x, y)$ denotes the depth of the pixel at position $(x, y)$ [S1].
3.  **Indicator Pixels:** A set of pixels $P_i$ identified as in-focus based on edge sharpness [S1].
4.  **Focus Frustums (FF):** For each image $i$, a volume in scene space defined by a near plane depth $n_i$ and a far plane depth $f_i$, bounded by the camera's view frustum [S1].
5.  **Refined Focus Frustums (RFF):** The updated frustums with near plane $\hat{n}_i$ and far plane $\hat{f}_i$ after refinement [S1].
6.  **Scene Points:** A set of high-density points $S$ sampled from the scene geometry [S1].

### Parameters
*   $\tau$: Threshold for the discrete Laplace operator to distinguish in-focus edges from out-of-focus regions. Typical value: 80 [S1].
*   $\epsilon$: Hyperparameter controlling the Focus Frustum Refinement step. It represents the minimum fraction of refined frustums that each sampled point in $S$ must be encompassed by. $\epsilon \in [0, 1.0]$. Typical value: 0.4 [S1]. $\epsilon = 0$ omits refinement; $\epsilon = 1$ implies no NeRF-FF application (training on all pixels) [S1].

### Steps

1.  **Preliminary Reconstruction:** Train a preliminary NeRF instance (e.g., Instant-NGP) on the blurry input images $\{I_i\}$ for 2,000 iterations to obtain per-view depth maps $\{D_i\}$ [S1].
2.  **Identify Indicator Pixels:** Compute the set of indicator pixels $P_i$ for each image $I_i$ by applying the discrete Laplace operator $\square$ to the grayscale version of $I_i$ and thresholding with $\tau$ [S1]:
    $$
    P_i = \{(x, y) \mid \square(I_i)(x, y) > \tau\}
    $$
    where $\square(I_i)(x, y)$ is the Laplace value of pixel $(x, y)$ [S1].
3.  **Initial Focus Frustum Estimation:** Sample the depth maps $D_i$ at the indicator pixels $P_i$ to estimate the focus depth and range (Depth of Field, DoF) using statistical analysis on a per-image basis [S1]. This provides initial near and far planes $n_i$ and $f_i$ for the Focus Frustums $FF_i$ [S1].
4.  **Focus Frustum Refinement:** Iteratively expand the near and far planes of the Focus Frustums to $RFF_i$ (with depths $\hat{n}_i, \hat{f}_i$) to ensure scene coverage [S1]. The refinement minimizes the accumulated depth change $R$ while ensuring that a fraction $\epsilon$ of randomly sampled high-density points $S$ are encompassed by at least an $\epsilon$ fraction of the refined frustums [S1]. The accumulated change $R$ is calculated only for frustums whose view frustum encompasses the spatial position $p \in S$ [S1]. The goal is to find $\hat{n}_i, \hat{f}_i$ such that:
    $$
    \text{minimize } R \quad \text{subject to } |\{i \mid p \in RFF_i\}| \ge \epsilon \cdot N \quad \forall p \in S
    $$
    (Note: The exact formulation of $R$ is not explicitly defined in the sources beyond "accumulated depth change" [S1].)
5.  **Mask Generation:** Reproject the dense scene points encompassed by the refined Focus Frustums into the respective image space to generate binary masks [S1]. For each pixel $(x, y)$ in image $i$, the mask is defined as:
    $$
    Mask_i(x, y) = \begin{cases} 
    1, & \text{if } D_i(x, y) \in [\hat{n}_i, \hat{f}_i] \\
    0, & \text{otherwise}
    \end{cases}
    $$
    Since all pixels in the depth map are visible, other boundaries of the Focus Frustum (side planes) do not need to be considered for the mask generation [S1].
6.  **Final NeRF Training:** Train the subsequent NeRF model (e.g., Instant-NGP, nerfacto, splatfacto) using the input images $\{I_i\}$ and the generated masks $\{Mask_i\}$ for 4,000 iterations [S1]. During training, only pixels $I_i(x, y)$ where $Mask_i(x, y) = 1$ are used for loss computation [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerff_generate_masks(input_images, tau=80, epsilon=0.4):
    """
    NeRF-FF Pipeline.
    
    Args:
        input_images: List of N images I_i, shape (H, W, 3), values [0, 255].
        tau: Threshold for Laplace operator.
        epsilon: Fraction of frustums covering each scene point.
        
    Returns:
        final_nerf_model: Trained NeRF model.
    """
    N = len(input_images)
    H, W = input_images[0].shape[:2]
    
    # Step 1: Preliminary Reconstruction
    # Train a preliminary NeRF (e.g., Instant-NGP) on input_images for 2000 iterations [S1]
    # This yields a scene representation from which depth maps are extracted.
    preliminary_nerf = train_preliminary_nerf(input_images, iterations=2000) # [Comment: External step per [S1]]
    depth_maps = [preliminary_nerf.get_depth_map(img) for img in input_images]

    # Step 2: Identify Indicator Pixels
    indicator_pixels = []
    for i, img in enumerate(input_images):
        gray = grayscale(img)
        laplace_vals = discrete_laplace(gray)
        P_i = {(x, y) for y in range(H) for x in range(W) if laplace_vals[y, x] > tau}
        indicator_pixels.append(P_i)

    # Step 3: Initial Focus Frustum Estimation
    initial_frustums = []
    for i in range(N):
        depths = [depth_maps[i][y, x] for (x, y) in indicator_pixels[i]]
        if not depths:
            n_i, f_i = 0.0, 10.0 # [G] Fallback for empty indicator set
        else:
            median_d = sum(depths) / len(depths) # [G] Simple mean as proxy for statistical analysis
            std_d = (sum((d - median_d)**2 for d in depths) / len(depths))**0.5
            n_i = max(0.0, median_d - 2.0 * std_d)
            f_i = median_d + 2.0 * std_d
        initial_frustums.append((n_i, f_i))

    # Step 4: Focus Frustum Refinement
    # Sample high-density points S from preliminary_nerf [S1]
    S = sample_scene_points(preliminary_nerf, count=10000) # [Comment: External step per [S1]]
    refined_frustums = []
    for i in range(N):
        n_hat, f_hat = refine_frustum(initial_frustums[i], S, epsilon, N)
        refined_frustums.append((n_hat, f_hat))

    # Step 5: Mask Generation
    masks = []
    for i in range(N):
        n_hat, f_hat = refined_frustums[i]
        mask = [[0]*W for _ in range(H)]
        for y in range(H):
            for x in range(W):
                if n_hat <= depth_maps[i][y, x] <= f_hat:
                    mask[y][x] = 1
        masks.append(mask)

    # Step 6: Final NeRF Training
    # Train subsequent NeRF on input_images using masks for 4000 iterations [S1]
    final_model = train_final_nerf(input_images, masks, iterations=4000) # [Comment: External step per [S1]]
    return final_model

def grayscale(image):
    """Convert RGB image to grayscale using standard luminance weights."""
    return 0.299 * image[:,:,0] + 0.587 * image[:,:,1] + 0.114 * image[:,:,2]

def discrete_laplace(gray_image):
    """Apply 3x3 discrete Laplace operator kernel."""
    H, W = gray_image.shape
    laplace = [[0.0]*W for _ in range(H)]
    for y in range(1, H-1):
        for x in range(1, W-1):
            laplace[y][x] = (4 * gray_image[y][x] 
                             - gray_image[y-1][x] - gray_image[y+1][x] 
                             - gray_image[y][x-1] - gray_image[y][x+1])
    return laplace

def sample_scene_points(nerf_model, count):
    """Sample points from the scene density field."""
    # [Comment: The source does not specify the exact sampling algorithm for S [S1]]
    return []

def refine_frustum(initial_frustum, S, epsilon, N):
    """
    Refines the near and far planes of a Focus Frustum.
    Minimizes accumulated depth change R while ensuring coverage.
    """
    n, f = initial_frustum
    # [Comment: The exact optimization algorithm for R is not detailed in the source [S1]]
    # The constraint requires each p in S to be covered by at least epsilon*N frustums.
    # This typically involves iterative expansion of n and f across all frustums.
    return n, f

def train_preliminary_nerf(images, iterations):
    # [Comment: External NeRF training step per [S1]]
    pass

def train_final_nerf(images, masks, iterations):
    # [Comment: External NeRF training step per [S1]]
    pass
```

### Edge Cases and Pitfalls
*   **Fully Blurred Images:** NeRF-FF is not suited for datasets containing fully blurred images [S1]. Low-quality dense reconstruction of the preliminary NeRF leads to inferior depth maps and floater artifacts (scene volume with high estimated optical density with no counterpart in the ground truth scene) [S1]. These artifacts can persist in the subsequent reconstruction, reducing image quality [S1].
*   **Threshold $\tau$:**
    *   Low $\tau$ values cause bad performance because weak signals (texture in out-of-focus regions) are considered indicators, leading to incorrect in-focus region identification [S1].
    *   High $\tau$ values cause slowly decreasing performance because sharp edges in in-focus regions are missed. However, the decline is small because the initial frustums are refined [S1].
*   **Dense Reconstruction Quality:** Mismatches between in-focus indicators and the spatial position of scene geometry cause erroneous depth estimates, negatively influencing initial Focus Frustum estimates [S1].
*   **Floater Artifacts:** Even with masking, NeRF-FF + iNGP can suffer from floater artifacts in the subsequent reconstruction, which reduce image quality depending on the camera position [S1].

## Operating characteristics
*   **Time Complexity:** NeRF-FF reduces training time by two orders of magnitude compared to state-of-the-art methods like PDRF, DP-NeRF, and Deblur-NeRF [S1]. Average training time is 45 seconds on an NVIDIA RTX 3090, encompassing both NeRF-FF execution (including preliminary NeRF training) and the subsequent Instant-NGP training [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:**
    *   On the real-world defocus dataset by Ma et al., NeRF-FF + iNGP achieves PSNR on par with SOTA methods and consistently outperforms them in SSIM by a fair margin, while trailing reference deblurring strategies slightly in LPIPS [S1].
    *   On the Wu et al. dataset, NeRF-FF + iNGP consistently produces better quality images than DoF-NeRF [S1].
    *   On synthetic datasets with fully blurred images, NeRF-FF underperforms SOTA due to low-quality dense reconstruction [S1].
*   **Compatibility:** Compatible with Instant-NGP, nerfacto, and splatfacto (3D Gaussian Splatting) [S1]. Applying NeRF-FF masks to nerfacto and splatfacto improves PSNR, SSIM, and LPIPS metrics with negligible impact on training duration [S1].
*   **Strengths:** High training speed, compatibility with static-ray NeRF variants, robustness to partial defocus blur.
*   **Limitations:** Poor performance on fully blurred images, potential for floater artifacts, dependency on the quality of the preliminary NeRF's depth estimation.

## Background
NeRF-FF was proposed by T. Wirth et al. in the paper "NeRF-FF: a plug-in method to mitigate defocus blur for runtime optimized neural radiance fields" [S1]. It addresses the limitations of existing defocus mitigation methods (e.g., PDRF, Deblur-NeRF) which rely on dynamic ray generation and are incompatible with performance-optimized NeRF variants like Instant-NGP [S1]. The method leverages the discrete Laplace operator to detect in-focus regions and estimates Focus Frustums to mask out-of-focus pixels during training [S1].

## Sources
[S1] NeRF-FF: a plug-in method to mitigate defocus blur for runtime optimized neural radiance fields, https://doi.org/10.1007/s00371-024-03507-y, fetched 2026-10-26
