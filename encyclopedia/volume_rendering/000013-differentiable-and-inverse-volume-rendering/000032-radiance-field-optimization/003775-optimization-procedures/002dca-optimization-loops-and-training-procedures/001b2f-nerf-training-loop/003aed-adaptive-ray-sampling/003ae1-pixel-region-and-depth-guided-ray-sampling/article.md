# Pixel Region and Depth Guided Ray Sampling


## Overview
Pixel Region and Depth Guided Ray Sampling is a ray sampling strategy that increases sampling probability in pixel areas exhibiting significant color and depth variation, reducing redundant rays in flat regions to accelerate network convergence and improve geometry perception [S1]. It sets itself apart from uniform sampling and other adaptive methods by explicitly leveraging both image color variance and estimated geometric depth boundaries to determine ray density, rather than relying solely on network-predicted density or opacity weights [S1].

## How it works
**Inputs:**
*   A set of $N$ source view images $I_i$ with height $H_i$ and width $W_i$ [S1].
*   A depth estimation module (specifically from the ENeRF framework) providing rough depth information and a cost volume for each pixel [S1].
*   Network parameters $\Theta$ for the radiance field [S1].

**Outputs:**
*   A set of sampled rays $\mathbb{R}_i$ for each image $I_i$, where the number of rays $M$ is distributed non-uniformly based on the sampling probability [S1].
*   Rendered colors $V_{\Theta}(r_i^k)$ for the sampled rays [S1].

**Data Structures:**
*   **Image Grid:** Each image $I_i$ is treated as a grid of pixels $(u, v)$ where $u \in [0, H_i]$ and $v \in [0, W_i]$ [S1].
*   **Depth Map:** A 2D array of depth values $d(u, v)$ estimated from the cost volume [S1].
*   **Color Map:** A 2D array of RGB color values $C(u, v)$ from the source image [S1].
*   **Sampling Probability Map:** A 2D array $P(u, v)$ storing the probability of selecting a ray for each pixel [S1].

**Parameters:**
*   $N$: The number of trained images (source views) [S1].
*   $M$: The total number of sampled rays per image [S1].
*   $\Theta$: The network parameters optimized by minimizing the color error [S1].

**Steps:**
1.  **Depth Estimation:** Obtain rough depth information $d(u, v)$ for each pixel $(u, v)$ in the source view images $I_i$ using the depth estimation module from the ENeRF framework [S1]. This involves leveraging the cost volume to compute depth for each pixel [S1].
2.  **Probability Calculation:** Calculate the sampling probability $P(u, v)$ for each pixel based on its color value $C(u, v)$ and the estimated depth information $d(u, v)$ [S1]. The method increases sampling probability in areas with significant color variation and regions where geometric depth varies markedly [S1]. [G: The specific mathematical formula for combining color variance and depth gradient into $P(u, v)$ is not provided in the sources; it is described qualitatively as being guided by these two factors.]
3.  **Ray Sampling:** During the ray sampling phase, select $M$ rays for each image $I_i$ such that the probability of selecting a ray from pixel $(u, v)$ is proportional to $P(u, v)$ [S1]. This results in sampling additional rays in pixel areas exhibiting substantial color variation and marked depth variation, while reducing rays in flat regions [S1].
4.  **Volume Rendering:** Perform volume rendering using the network $F_{\Theta}$ to output colors and densities along the sampled rays $r_i^k \in \mathbb{R}_i$ [S1]. The volume rendering function $V_{\Theta}$ computes the predicted color $\hat{c}_i^k$ for each ray [S1].
5.  **Optimization:** Optimize the network parameters $\Theta$ by minimizing the volume rendering loss [S1]:
    $$ \min_{\Theta} \mathcal{L} = \frac{1}{NM} \sum_{r_i^k \in \mathbb{R}_i} \|V_{\Theta}(r_i^k) - c_i^k\|_2^2 $$
    where $c_i^k$ is the true color of the pixel corresponding to ray $r_i^k$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixel_region_depth_guided_sampling(source_images, depth_estimator, num_rays_per_image):
    """
    source_images: List of N images, each with height H and width W.
    depth_estimator: Function/module that returns depth map and cost volume from source images.
    num_rays_per_image: M, the total number of rays to sample per image.
    """
    sampled_rays = []
    
    for image in source_images:
        # Step 1: Obtain depth information
        depth_map = depth_estimator.get_depth_map(image)
        color_map = image.get_colors()
        
        # Step 2: Calculate sampling probability
        # Note: The exact combination of color variation and depth variation 
        # is not specified in the source. [G] Placeholder for the specific 
        # probability calculation logic.
        probability_map = calculate_sampling_probability(color_map, depth_map)
        
        # Normalize probabilities to sum to 1
        total_prob = sum(probability_map.flatten())
        normalized_prob = probability_map / total_prob
        
        # Step 3: Sample M rays based on probability
        # Flatten the probability map to create a 1D distribution
        flat_probs = normalized_prob.flatten()
        
        # Sample M indices from the flattened grid
        # Using inverse transform sampling or categorical sampling
        selected_indices = sample_indices(flat_probs, num_rays_per_image)
        
        # Convert indices back to (u, v) coordinates
        rays_for_image = []
        for idx in selected_indices:
            u, v = index_to_pixel_coords(idx, image.height, image.width)
            ray = create_ray(u, v, image.camera_params)
            rays_for_image.append(ray)
            
        sampled_rays.append(rays_for_image)
        
    return sampled_rays

def calculate_sampling_probability(color_map, depth_map):
    """
    Placeholder for the specific method of combining color and depth 
    information to determine sampling probability.
    """
    # [G] Implementation detail missing from source. 
    # Typically involves computing gradients or variance for color and depth.
    pass
```

**Edge Cases and Pitfalls:**
*   **Depth Estimation Errors:** The method relies on the accuracy of the depth estimation module from ENeRF. Inaccurate depth estimates in textureless or reflective regions may lead to suboptimal ray sampling [S1].
*   **Probability Calculation:** The specific formula for combining color and depth information into the sampling probability is not provided in the source [S1]. Implementers must define this combination, potentially using gradient magnitudes or variance metrics.
*   **Uniform Sampling Baseline:** The method contrasts with uniform sampling where $r_i(u, v) \sim U(I_i)$ [S1]. Deviations from this uniform distribution must be carefully managed to ensure sufficient coverage of the scene.

## Operating characteristics
*   **Time Complexity:** The method aims to accelerate network convergence by reducing redundant rays, potentially lowering the number of necessary network evaluations per iteration compared to uniform sampling [S1]. However, the overhead of depth estimation and probability calculation adds computational cost per iteration [S1].
*   **Memory Complexity:** Requires storing depth maps and probability maps for each image, in addition to the standard radiance field parameters [S1].
*   **Performance:** Applied to the ENeRF framework, this method achieves state-of-the-art performance on view synthesis and training time [S1]. It enables faster per-scene optimization, achieving the same convergence effect in minutes through fine-tuning [S1].
*   **Strengths:** Improves geometry perception, especially for rich-texture regions, by focusing samples on areas with significant color and depth variations [S1].
*   **Limitations:** Relies on the availability and accuracy of a depth estimation module (e.g., from ENeRF) [S1]. May not perform as well in scenes where depth estimation is unreliable.
*   **When to Use:** Suitable for scenes with complex textures and geometry where accurate depth estimation is possible, and where accelerating training convergence is a priority [S1].
*   **When Not to Use:** Not suitable for scenes where depth estimation is difficult or unreliable, or when computational resources for depth estimation are limited [S1].

## Background
*   **Origin:** The method was introduced in the paper "Efficient ray sampling for radiance fields reconstruction" [S1].
*   **History:** It builds upon the ENeRF framework [S1] and addresses the inefficiency of uniform ray sampling in NeRF [S1]. The authors observed that uniform sampling fails to capture the non-uniform distribution of color differences and scene density [S1].
*   **Usage:** Applied to the ENeRF framework for novel view synthesis and training acceleration [S1].

## Sources
[S1] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
