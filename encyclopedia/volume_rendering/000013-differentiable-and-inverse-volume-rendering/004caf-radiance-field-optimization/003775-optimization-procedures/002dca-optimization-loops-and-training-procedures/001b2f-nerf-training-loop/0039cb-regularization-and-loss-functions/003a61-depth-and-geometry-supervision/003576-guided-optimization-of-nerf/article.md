# Guided Optimization of NeRF

Also called: Confidence-Based Depth Filtering

## Overview
Guided Optimization of NeRF mitigates shape-radiance ambiguity in poorly textured indoor scenes by constraining the sampling range of Neural Radiance Fields (NeRF) using adapted monocular depth priors [S1]. Unlike standard NeRF, which samples uniformly between near and far bounds derived from sparse 3D reconstruction, this method calculates adaptive sampling intervals based on multi-view geometric consistency errors, ensuring sampled points concentrate around reliable depth estimates [S1]. It sets itself apart from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by using depth priors primarily to guide the *sampling strategy* during optimization rather than solely as a direct regression target, thereby avoiding the degradation caused by unreliable flow correspondences in texture-less regions [S1].

## How it works
**Inputs:**
- $N$ input views with RGB images and camera poses.
- Adapted monocular depth priors $\{D^i\}_{i=1}^N$ for each view, obtained from a finetuned monocular depth network [S1].
- Camera intrinsics $K$ and relative poses $T^{i \rightarrow j}$ [S1].

**Outputs:**
- Optimized NeRF model $\mathcal{F}_\theta: (\mathbf{x}, \mathbf{d}) \rightarrow (c, \sigma)$ representing implicit radiance fields [S1].
- Refined depth map for each view.

**Data Structures:**
- **Error Map ($e_i$):** A 2D array per view $i$, storing the average of the top $K$ minimum cross-view reprojection errors for each pixel [S1].
- **Confidence Map ($S^i_j$):** A 2D array per view $i$, storing confidence scores for each pixel $j$ [S1].
- **Sampling Bounds ($t_n, t_f$):** Scalar values per ray, defining the near and far limits for stratified sampling [S1].

**Parameters:**
- $K=4$: Number of minimum cross-view depth projection errors used to calculate the error map value [S1].
- $\alpha_l = 0.05$: Lower bound for the relative sampling range expansion [S1].
- $\alpha_h = 0.15$: Higher bound for the relative sampling range expansion [S1].
- $M$: Number of sample points per ray (standard NeRF parameter) [S1].

**Steps:**

1.  **Acquire Adapted Depth Priors:** Obtain initial depth maps $\{D^i\}_{i=1}^N$ using a monocular depth network finetuned on the target scene [S1].

2.  **Compute Error Maps:**
    For each view $i$ and every other view $j$:
    a. Project the depth map $D^i$ to view $j$ using camera intrinsics $K$ and relative pose $T^{i \rightarrow j}$ to obtain projected coordinates $p^{i \rightarrow j}$ and projected depth $D^{i \rightarrow j}$ [S1].
    b. Sample the ground truth prior $D^j$ at coordinates $p^{i \rightarrow j}$ to get $D^{j'}$ [S1].
    c. Calculate the relative reprojection error between $D^{j'}$ and $D^{i \rightarrow j}$ [S1].
    d. For each pixel in view $i$, collect the reprojection errors from all other views. Select the $K$ minimum errors and compute their average to form the error map value $e_i$ for that pixel [S1].

3.  **Calculate Adaptive Sample Ranges:**
    For each ray $\mathbf{r}$ corresponding to a pixel in view $i$ with adapted depth prior $D$ and error map value $e$:
    a. Clamp the error $e$ to the interval $[\alpha_l, \alpha_h]$ [S1].
    b. Compute the near bound $t_n$ and far bound $t_f$ using the formulas:
       $$ t_n = D(1 - \text{clamp}(e, \alpha_l, \alpha_h)) $$
       $$ t_f = D(1 + \text{clamp}(e, \alpha_l, \alpha_h)) $$
       Here, $D$ is the adapted depth prior value for the ray [S1].

4.  **Stratified Sampling:**
    Partition the interval $[t_n, t_f]$ into $M$ bins. For each bin $i \in [1, M]$, sample a point $t_i$ uniformly from the bin interval:
    $$ t_i \sim \mathcal{U} \left[ t_n + \frac{i-1}{M}(t_f - t_n), \,\, t_n + \frac{i}{M}(t_f - t_n) \right] $$
    Evaluate the NeRF network $\mathcal{F}_\theta$ at positions $\mathbf{x}_i$ along the ray at depths $t_i$ to obtain color $c_i$ and density $\sigma_i$ [S1].

5.  **Volume Rendering:**
    Compute the accumulated transmittance $T_i = \exp \left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $\delta_i = t_{i+1} - t_i$ [S1].
    Render the RGB color $C(\mathbf{r})$ and depth $D(\mathbf{r})$ for the ray:
    $$ C(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
    $$ D(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) t_i $$
    [S1]

6.  **Compute Confidence Scores:**
    For each pixel $j$ in view $i$, calculate the confidence score $S^i_j$ based on the $L_1$ difference between the ground truth RGB image $C_{gt}^i(j)$ and the rendered RGB image $C_{render}^i(j)$, normalized by 255:
    $$ S^i_j = 1 - \frac{1}{3} || C_{gt}^i(j) - C_{render}^i(j) ||_1 $$
    The $L_1$ norm is taken over the 3 color channels [S1].

7.  **Depth Refinement (Post-Processing):**
    Apply plane bilateral filtering to the rendered depth map, weighted by the confidence scores $S^i_j$, to produce the final refined depth output [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def guided_optimization_nerf(views, adapted_depths, K=4, alpha_l=0.05, alpha_h=0.15, M=48):
    """
    views: List of view objects containing RGB images, intrinsics K, and poses.
    adapted_depths: List of depth maps corresponding to views.
    """
    N = len(views)
    error_maps = []
    
    # Step 2: Compute Error Maps
    for i in range(N):
        view_i = views[i]
        depth_i = adapted_depths[i]
        h, w = depth_i.shape
        err_map = np.zeros((h, w))
        
        # Collect reprojection errors from all other views
        all_errors = []
        for j in range(N):
            if i == j: continue
            view_j = views[j]
            depth_j = adapted_depths[j]
            
            # Project depth from i to j
            # proj function handles K, T^{i->j}, and depth_i
            p_ij, D_ij = project_depth(depth_i, view_i.K, view_i.T_to(view_j))
            
            # Sample depth_j at projected coordinates
            # bilinear_sample handles interpolation and boundary checks
            D_j_prime = bilinear_sample(depth_j, p_ij)
            
            # Calculate relative error
            # Avoid division by zero
            valid_mask = D_j_prime > 0
            relative_error = np.abs(D_ij - D_j_prime) / (D_j_prime + 1e-6)
            
            # Store errors for valid pixels
            errors = relative_error[valid_mask]
            all_errors.append(errors)
            
        # For each pixel, average the top K minimum errors across views
        # Note: The source describes calculating e_i as the average of top K min errors.
        # Implementation detail: We aggregate errors per pixel across views, 
        # then take the K smallest.
        for y in range(h):
            for x in range(w):
                # Get errors at (x,y) from all views
                pixel_errors = []
                for j in range(N):
                    if i == j: continue
                    # Re-calculate or retrieve stored projection errors for (x,y)
                    # Assuming we stored them or re-projected
                    p_ij, D_ij = project_depth_at_pixel(depth_i, view_i.K, view_i.T_to(views[j]), x, y)
                    D_j_prime = bilinear_sample_at_pixel(adapted_depths[j], views[j], p_ij)
                    if D_j_prime > 0:
                        err = np.abs(D_ij - D_j_prime) / D_j_prime
                        pixel_errors.append(err)
                
                if len(pixel_errors) >= K:
                    # Sort and take K smallest
                    pixel_errors.sort()
                    err_map[y, x] = np.mean(pixel_errors[:K])
                else:
                    err_map[y, x] = 1.0 # Default high error if insufficient data
                    
        error_maps.append(err_map)

    # Step 3 & 4: Sampling and Rendering Loop (during training)
    # This function would be called per batch/iteration
    def get_sample_bounds(view_idx, pixel_coords):
        i = view_idx
        x, y = pixel_coords
        D = adapted_depths[i][y, x]
        e = error_maps[i][y, x]
        
        # Clamp error
        e_clamped = np.clip(e, alpha_l, alpha_h)
        
        t_n = D * (1 - e_clamped)
        t_f = D * (1 + e_clamped)
        
        return t_n, t_f

    def stratified_sample(t_n, t_f, M):
        bins = np.linspace(t_n, t_f, M + 1)
        samples = np.random.uniform(bins[:-1], bins[1:])
        return samples

    # Step 6 & 7: Confidence and Filtering (after training/inference)
    def compute_confidence(view_idx, gt_rgb, rendered_rgb):
        i = view_idx
        # Normalize by 255
        gt_norm = gt_rgb / 255.0
        render_norm = rendered_rgb / 255.0
        
        # L1 norm over channels
        l1_diff = np.mean(np.abs(gt_norm - render_norm), axis=-1) # Assuming H,W,C
        confidence = 1.0 - l1_diff
        return confidence

    def refine_depth(depth_map, confidence_map):
        # Apply plane bilateral filtering weighted by confidence
        # Using an off-the-shelf implementation or standard bilateral filter
        # with confidence as the weight map
        refined = plane_bilateral_filter(depth_map, weight_map=confidence_map)
        return refined

    return error_maps
```

**Edge Cases and Pitfalls:**
- **Shape-Radiance Ambiguity:** In poorly textured indoor areas (e.g., walls), NeRF may synthesize high-quality RGB images with incorrect geometry. Standard sampling fails here because sampled points predict similar RGB values with confidence concentrated in a limited range, leading to depth deviation from ground truth [S1]. The guided optimization addresses this by explicitly limiting the sampling range around depth priors [S1].
- **Correspondence Degradation:** Applying existing correspondence-based neural optimization (like CVD) to adapted depth priors can degrade quality due to unreliable flow correspondences in poorly textured regions [S1].
- **Precision:** The clamping of error $e$ to $[\alpha_l, \alpha_h]$ prevents the sampling range from becoming too narrow (if error is very low) or too wide (if error is very high), balancing diversity and precision [S1].

## Subtopics

### [ENeRF with Guided Sampling Training](7580-enerf-with-guided-sampling-training.md)
Integrates pixel-region and depth-boundary-guided ray sampling into the Efficient NeRF framework. It sets itself apart by actively guiding sampling locations using adapted monocular depth priors to mitigate shape-radiance ambiguity while reducing GPU memory consumption compared to standard ENeRF. Choose this when memory efficiency is critical for generalizable NeRF models.

## Operating characteristics
- **Time Complexity:** The error map computation requires $O(N^2 \cdot H \cdot W)$ operations for $N$ views of resolution $H \times W$ due to cross-view reprojection checks [S1]. The training loop complexity is similar to standard NeRF but with potentially fewer effective samples if the adaptive range is narrow, though the overhead of error map calculation adds to preprocessing time [S1].
- **Memory Complexity:** Requires storing $N$ depth maps and $N$ error maps, plus the standard NeRF model weights [S1].
- **Measured Performance:** On the ScanNet dataset, the method outperforms state-of-the-art depth estimation methods in all metrics, significantly outperforming original NeRF in view synthesis quality and depth accuracy [S1].
- **Strengths:** Effectively mitigates shape-radiance ambiguity in texture-less indoor scenes; improves depth accuracy without requiring ground-truth depth supervision during training; leverages strong monocular priors.
- **Limitations:** Relies on the quality of adapted monocular depth priors; may degrade if priors are inaccurate; not explicitly designed for outdoor scenes with rich textures where standard NeRF assumptions hold [S1].
- **When to use:** Indoor multi-view stereo tasks with texture-less regions where standard NeRF fails to recover correct geometry.
- **When not to use:** Outdoor scenes with rich textures where standard NeRF sampling is sufficient, or when computational resources for cross-view reprojection are severely limited.

## Background
The method was introduced in "Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo" [S1]. It builds on the original NeRF proposal [33] and addresses the shape-radiance ambiguity problem identified in [61], which posits that while correct shapes are favored for their smoother surface light fields, NeRF can still fit incorrect geometry in texture-less areas. The authors observed that standard NeRF struggles in poorly textured indoor environments and proposed using adapted monocular depth priors to guide the sampling process, thereby constraining the search space for the implicit volume [S1].

## Sources
[S1] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
