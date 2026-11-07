# Pixel Region and Depth Guided Ray Sampling


## Overview
Pixel Region and Depth Guided Ray Sampling is a ray selection strategy that increases sampling probability in pixel areas exhibiting significant color and depth variation, thereby reducing redundant rays in flat regions to accelerate network convergence and improve geometry perception [S1]. Unlike conventional NeRF which uses uniform distribution, this method samples varying ray densities based on depth boundaries and pixel regions to simultaneously accelerate convergence and improve geometry perception [S1]. It is distinct from sibling methods like [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md) which focuses on angular resolution, and [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) which focuses on per-ray depth refinement, by operating at the image-space level to select *which* rays to cast based on 2D image statistics and estimated depth maps [S1].

## How it works
**Inputs:**
*   A set of $N$ training source images $I_i$ with known camera poses.
*   A depth estimation module (specifically from the ENeRF framework) capable of producing a rough depth map for each source view [S1].
*   The volume rendering network parameters $\Theta$ [S1].

**Outputs:**
*   A set of $M$ sampled rays per image, $\mathbb{R}_i$, where the spatial distribution of rays is non-uniform, biased towards regions of high color variance and depth discontinuity [S1].
*   Rendered colors and densities for these rays [S1].

**Data Structures:**
*   **Image $I_i$**: A 2D array of size $H_i \times W_i \times 3$ representing color values [S1].
*   **Depth Map $D_i$**: A 2D array of size $H_i \times W_i$ representing estimated depth values for each pixel $(u, v)$ [S1].
*   **Sampling Probability Map $P_i$**: A 2D array of size $H_i \times W_i$ where $P_i(u, v)$ represents the probability of selecting the ray originating from pixel $(u, v)$ [S1].
*   **Ray Set $\mathbb{R}_i$**: A list of $M$ rays, where each ray is defined by its origin (pixel center) and viewing direction [S1].

**Parameters:**
*   $N$: The number of trained images (source views) [S1].
*   $M$: The number of sampled rays per image [S1].
*   $\Theta$: The network parameters optimized by minimizing the color error [S1].
*   Typical values for $M$ are not explicitly given in the sources, but the method aims to reduce redundant rays to accelerate training [S1].

**Steps:**

1.  **Depth Estimation**: For each training image $I_i$, obtain a rough depth map $D_i$ using the depth estimation module from the ENeRF framework [S1]. This module leverages the cost volume to obtain depth information for each pixel [S1].

2.  **Probability Calculation**: Calculate the sampling probability $P_i(u, v)$ for each pixel $(u, v)$ in image $I_i$. This probability is derived from two factors:
    *   **Color Variation**: Pixels with substantial color variation from source view images are assigned higher probability [S1].
    *   **Depth Variation**: Pixels where geometric depth varies markedly are assigned higher probability [S1].
    
    The exact mathematical formula for combining color and depth variance into $P_i(u, v)$ is not provided in the sources [S1]. The sources state that the probability is obtained using the color value and the estimated depth information [S1]. [G] In practice, this typically involves computing the gradient magnitude of the color image and the depth map, normalizing them, and summing or weighting them to form a composite importance map, which is then normalized to sum to 1.

3.  **Ray Sampling**: During the ray sampling phase, select $M$ rays for image $I_i$ based on the probability map $P_i$.
    *   Unlike uniform sampling where $r_i(u, v) \sim U(I_i)$ with $u \in [0, H_i]$ and $v \in [0, W_i]$ [S1], this method samples additional rays in pixel areas exhibiting substantial color variation [S1].
    *   Similarly, guided by the depth-based sampling strategy, more rays are sampled in regions where geometric depth varies markedly [S1].
    *   This is typically implemented using inverse transform sampling or stochastic selection based on the discrete probability distribution defined by $P_i$ [S1].

4.  **Volume Rendering**: Perform volume rendering using the network to output colors and densities along the sampled rays [S1]. The network $F_\Theta: (p, d) \rightarrow (c, \sigma)$ maps 3D coordinates and viewing directions to volume density $\sigma$ and directional emitted color $c$ [S1]. The volume rendering function $V_\Theta$ predicts the color $\hat{c}_i^k$ for each ray $r_i^k$ [S1].

5.  **Optimization**: Optimize the network parameters $\Theta$ by minimizing the volume rendering loss [S1]. The optimization goal is:
    $$ \min_{\Theta} \mathcal{L} = \frac{1}{NM} \sum_{r_i^k \in \mathbb{R}_i} \|V_{\Theta}(r_i^k) - c_i^k\|_2^2 $$
    where $c_i^k$ is the true color of pixels, $M$ is the number of sampled rays per image, $N$ is the number of trained images, and $r_i^k$ is the $k$-th ray from the $i$-th training image [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixel_region_depth_guided_sampling(images, depth_maps, num_rays_per_image, camera_intrinsics):
    """
    Samples rays for a batch of images based on color and depth variance.
    
    Args:
        images: List of N images, each of shape (H, W, 3).
        depth_maps: List of N depth maps, each of shape (H, W).
        num_rays_per_image: M, the number of rays to sample per image.
        camera_intrinsics: Camera parameters to convert pixel coordinates to rays.
        
    Returns:
        rays: List of lists, where rays[i] contains M rays for image i.
    """
    all_rays = []
    
    for i in range(len(images)):
        img = images[i]
        depth = depth_maps[i]
        H, W, _ = img.shape
        
        # Step 1: Calculate Color Variance
        # [G] Compute gradient magnitude of the color image as a proxy for color variation.
        # Using Sobel operators or simple finite differences.
        color_grad = compute_gradient_magnitude(img)
        
        # Step 2: Calculate Depth Variance
        # [G] Compute gradient magnitude of the depth map as a proxy for depth variation.
        depth_grad = compute_gradient_magnitude(depth)
        
        # Step 3: Combine into Sampling Probability Map
        # [G] Normalize gradients to [0, 1] and sum them.
        # The exact weighting between color and depth is not specified in sources [S1].
        # We assume equal weighting or a simple sum.
        importance_map = normalize(color_grad) + normalize(depth_grad)
        
        # Normalize the importance map to form a probability distribution
        total_importance = np.sum(importance_map)
        if total_importance > 0:
            prob_map = importance_map / total_importance
        else:
            prob_map = np.ones((H, W)) / (H * W)
            
        # Step 4: Sample M pixels based on probability map
        # Flatten the probability map
        flat_probs = prob_map.flatten()
        
        # Sample M indices from the flattened map
        # Using numpy's random.choice with probabilities
        sampled_indices = np.random.choice(H * W, size=num_rays_per_image, p=flat_probs)
        
        # Convert flat indices back to (u, v) coordinates
        u_coords = sampled_indices // W
        v_coords = sampled_indices % W
        
        # Step 5: Generate rays from sampled pixels
        rays = []
        for u, v in zip(u_coords, v_coords):
            # [G] Convert pixel (u, v) to ray origin and direction using camera intrinsics.
            # This is a standard pinhole camera model operation.
            ray_origin, ray_direction = pixel_to_ray(u, v, camera_intrinsics[i])
            rays.append((ray_origin, ray_direction))
            
        all_rays.append(rays)
        
    return all_rays

def compute_gradient_magnitude(image):
    """
    Computes the gradient magnitude of an image.
    """
    # [G] Simple implementation using Sobel filters
    # For color images, compute gradient for each channel and sum or take max
    # For depth, compute gradient on the single channel
    pass

def normalize(array):
    """
    Normalizes an array to [0, 1].
    """
    min_val = np.min(array)
    max_val = np.max(array)
    if max_val - min_val == 0:
        return np.zeros_like(array)
    return (array - min_val) / (max_val - min_val)

def pixel_to_ray(u, v, intrinsics):
    """
    Converts pixel coordinates to ray origin and direction.
    """
    # [G] Standard pinhole camera projection inversion
    pass
```

**Edge Cases and Pitfalls:**
*   **Depth Estimation Errors**: The method relies on the accuracy of the depth estimation module from ENeRF. If the depth map is inaccurate, the sampling may be biased towards incorrect regions, potentially harming geometry perception [S1].
*   **Flat Regions**: In regions with no color or depth variation, the probability map may be uniform or near-zero, leading to potential under-sampling if not handled carefully [S1].
*   **Precision**: The calculation of gradients and normalization should be done with sufficient precision to avoid numerical instability, especially in regions with very low variance [S1].

## Operating characteristics
*   **Time Complexity**: The sampling step involves computing gradients and sampling, which is $O(H \times W)$ per image, where $H$ and $W$ are image dimensions. This is generally faster than per-ray density estimation methods that require network queries for every potential sample point.
*   **Memory Complexity**: Requires storing the depth maps and probability maps, which is $O(H \times W)$ per image [S1].
*   **Measured Performance**: Applied to the ENeRF framework, this method achieves state-of-the-art performance on both view synthesis and training time [S1]. It enables faster per-scene optimization, achieving the same convergence effect in minutes through fine-tuning reconstructions from given views [S1].
*   **Strengths**: Accelerates network convergence by reducing redundant rays in flat regions; improves geometry perception by focusing on depth boundaries; can be directly transferred to existing NeRF models [S1].
*   **Limitations**: Relies on the quality of the external depth estimation module; may not adapt as dynamically to local density changes as per-ray methods like [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) [S1].
*   **When to use**: When training time is a bottleneck and the scene has distinct texture and depth boundaries; when using frameworks like ENeRF that provide depth estimation [S1].
*   **When not to use**: When high-fidelity geometry is required in regions with low color/depth variance but high density (e.g., transparent objects); when an accurate depth estimator is not available.

## Background
*   **Origin**: Introduced in the paper "Efficient ray sampling for radiance fields reconstruction" [S1].
*   **History**: The method builds upon the ENeRF framework [15] and addresses the inefficiency of uniform ray sampling in NeRF [18]. It was developed to accelerate training and improve geometry perception by leveraging pixel-level statistics and estimated depth [S1].
*   **Usage**: Applied to the ENeRF framework for view synthesis and training time acceleration [S1].

## Sources
[S1] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
