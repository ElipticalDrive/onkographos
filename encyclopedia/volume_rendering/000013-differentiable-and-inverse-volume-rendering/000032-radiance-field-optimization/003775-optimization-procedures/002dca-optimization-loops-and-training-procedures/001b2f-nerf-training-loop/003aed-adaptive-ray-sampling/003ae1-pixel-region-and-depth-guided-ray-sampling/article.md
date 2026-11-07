# Pixel Region and Depth Guided Ray Sampling


## Overview
Pixel Region and Depth Guided Ray Sampling is a ray sampling strategy that increases sampling probability in pixel areas exhibiting significant color and depth variation, reducing redundant rays in flat regions to accelerate network convergence and improve geometry perception [S1]. Unlike conventional NeRF which uses uniform distribution, this method samples varying ray densities based on depth boundaries and pixel regions to simultaneously accelerate convergence and improve geometry perception [S1]. It is distinct from sibling methods like [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) which refines points along a single ray, and [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) which uses spatial data structures for hierarchical sampling; instead, it relies on 2D image-space cues (color variance and estimated depth) to bias the selection of which rays are cast from the source views.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ source training images $I_i$ with known camera poses; the ENeRF framework's cost volume and depth estimation module [S1]; the radiance field network parameters $\Theta$.
*   **Outputs:** A set of sampled rays $\mathbb{R}_i$ per image, and the optimized network parameters $\Theta$ minimizing the rendering loss.

**Data Structures:**
*   **Cost Volume:** A 3D tensor $C(u, v, d)$ derived from multi-view stereo, where $u, v$ are pixel coordinates and $d$ is disparity/depth [S1].
*   **Depth Map:** A 2D array $D(u, v)$ representing the estimated rough depth for each pixel, obtained from the cost volume [S1].
*   **Color Image:** The source image $I_i$ with pixel values $C_{pixel}(u, v)$ [S1].

**Steps:**
1.  **Depth Estimation:** Obtain rough depth information $D(u, v)$ for each pixel $(u, v)$ in the source view images using the depth estimation module from the ENeRF framework [S1]. This involves leveraging the cost volume to determine the most likely depth for each pixel [S1].
2.  **Probability Calculation:** Calculate a sampling probability $P(u, v)$ for each pixel $(u, v)$ based on two factors:
    *   **Color Variation:** Measure the local color variance or gradient magnitude in the source image $I_i$ at $(u, v)$. Pixels with substantial color variation receive higher probability [S1].
    *   **Depth Variation:** Measure the local depth variance or gradient in the estimated depth map $D(u, v)$ at $(u, v)$. Pixels where geometric depth varies markedly receive higher probability [S1].
    *   The combined probability $P(u, v)$ is proportional to the sum or product of these variation metrics. (Note: The exact normalization formula for $P(u, v)$ is not explicitly provided in the sources, but it must sum to 1 over all pixels in the image to serve as a probability distribution [S1]).
3.  **Ray Selection:** Sample $M$ rays per image $I_i$ by drawing pixel coordinates $(u, v)$ from the probability distribution $P(u, v)$. This results in additional rays being sampled in pixel areas exhibiting substantial color variation and marked depth variation [S1].
4.  **Volume Rendering:** For each selected ray $r_i^k$, sample points along the ray segment $[t_{near}, t_{far}]$ and evaluate the radiance field network $F_\Theta$ to obtain density $\sigma$ and color $c$. Perform volume rendering to compute the predicted pixel color $\hat{c}_i^k$ [S1].
5.  **Optimization:** Update network parameters $\Theta$ by minimizing the volume rendering loss [S1]:
    $$ \min_{\Theta} \mathcal{L} = \frac{1}{NM} \sum_{r_i^k \in \mathbb{R}_i} \|V_{\Theta}(r_i^k) - c_i^k\|_2^2 $$
    where $c_i^k$ is the true color of the pixel corresponding to ray $r_i^k$, $M$ is the number of sampled rays per image, $N$ is the number of trained images, and $V_{\Theta}$ is the volume rendering function [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixel_region_depth_guided_sampling(source_images, depth_maps, num_rays_per_image):
    """
    source_images: List of N images, each of shape (H, W, 3)
    depth_maps: List of N depth maps, each of shape (H, W)
    num_rays_per_image: M, number of rays to sample per image
    Returns: List of sampled ray origins and directions for each image
    """
    all_sampled_rays = []
    
    for i in range(len(source_images)):
        img = source_images[i]
        depth = depth_maps[i]
        H, W = img.shape[0], img.shape[1]
        
        # Step 1: Calculate Color Variation (Gradient Magnitude)
        # Compute gradient magnitude in the image
        # Using Sobel operators or similar for gradient estimation
        gray_img = rgb_to_grayscale(img)
        grad_x, grad_y = compute_gradients(gray_img)
        color_variance = sqrt(grad_x**2 + grad_y**2)
        
        # Step 2: Calculate Depth Variation (Depth Gradient Magnitude)
        # Compute gradient magnitude in the depth map
        depth_grad_x, depth_grad_y = compute_gradients(depth)
        depth_variance = sqrt(depth_grad_x**2 + depth_grad_y**2)
        
        # Step 3: Combine Variations to form Sampling Probability
        # The sources state probability is based on color value and estimated depth information.
        # We combine the two variance maps. A simple sum is a standard approach for such heuristics.
        # Normalize to sum to 1 to create a valid probability distribution.
        combined_variance = color_variance + depth_variance
        total_variance = sum(combined_variance.flatten())
        
        if total_variance == 0:
            # Fallback to uniform sampling if no variation is detected
            probabilities = ones(H * W) / (H * W)
        else:
            probabilities = combined_variance.flatten() / total_variance
            
        # Step 4: Sample M pixels based on probabilities
        # Draw M indices from the range [0, H*W) using the probability distribution
        sampled_indices = sample_without_replacement(probabilities, num_rays_per_image)
        
        # Convert flat indices to (u, v) coordinates
        sampled_u, sampled_v = indices_to_coordinates(sampled_indices, W)
        
        # Generate rays from these pixel coordinates
        # Ray origin and direction are determined by camera pose of image i and pixel (u, v)
        rays = generate_rays_from_pixels(sampled_u, sampled_v, camera_pose_i)
        
        all_sampled_rays.append(rays)
        
    return all_sampled_rays
```

**Parameters:**
*   $N$: The number of trained images (source views) [S1].
*   $M$: The number of sampled rays per image [S1].
*   $\Theta$: The network parameters optimized by minimizing the color error [S1].
*   Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Zero Variance:** If an image region is completely flat in both color and depth, the probability distribution may be undefined or uniform. A fallback to uniform sampling or a small epsilon addition to variance maps is recommended [G].
*   **Depth Estimation Errors:** The method relies on the accuracy of the rough depth information from ENeRF. Incorrect depth estimates in occluded or textureless regions may lead to suboptimal sampling [S1].
*   **Boundary Conditions:** The gradient calculations at image boundaries must handle edge pixels correctly (e.g., using padding or ignoring boundaries) [G].

## Operating characteristics
*   **Time Complexity:** The sampling step requires computing gradients for color and depth maps, which is $O(H \times W)$ per image. Sampling $M$ rays is $O(M \log(H \times W))$ or $O(M)$ with alias sampling. This is comparable to uniform sampling in terms of ray generation overhead, but reduces the number of network evaluations needed for convergence.
*   **Memory Complexity:** Requires storing the cost volume and depth maps from ENeRF, which adds memory overhead compared to standard NeRF [S1].
*   **Measured Performance:** Applied to the ENeRF framework, it achieves state-of-the-art performance on view synthesis and training time, enabling same convergence effect in minutes through fine-tuning reconstructions from given views [S1].
*   **Strengths:** Accelerates network convergence by focusing computational resources on informative regions (high color/depth variation) [S1]. Improves geometry perception, especially for rich-texture regions [S1].
*   **Limitations:** Relies on the quality of the initial depth estimation from ENeRF. May not be as effective in scenes with uniform color and depth (e.g., blank walls) where variation is low [S1].
*   **When to use:** When training time is a constraint and the scene has significant texture and geometric detail. Suitable for static or dynamic scenes where ENeRF's depth estimation is reliable [S1].
*   **When not to use:** In scenes with very low texture or uniform depth where the sampling probability distribution provides little guidance.

## Background
The method was introduced in the paper "Efficient ray sampling for radiance fields reconstruction" [S1]. It builds on the ENeRF framework [15] and addresses the inefficiency of uniform ray sampling in NeRF, which fails to match the real-world surface density distribution [S1]. The key hypothesis is that redundant rays in flat regions can be reduced to accelerate fitting, while increasing sampling in areas with significant color and depth variations improves efficiency and geometry perception [S1].

## Sources
[S1] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
