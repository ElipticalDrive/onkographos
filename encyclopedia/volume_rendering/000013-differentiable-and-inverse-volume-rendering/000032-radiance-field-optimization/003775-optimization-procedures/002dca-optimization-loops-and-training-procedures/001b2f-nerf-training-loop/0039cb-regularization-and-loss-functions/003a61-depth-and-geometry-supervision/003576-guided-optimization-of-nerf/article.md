# Guided Optimization of NeRF

Also called: Confidence-Based Depth Filtering

## Overview
Guided Optimization of NeRF mitigates shape-radiance ambiguity in indoor scenes by restricting the sampling range of points along camera rays using adapted monocular depth priors [S1]. Unlike standard NeRF, which samples uniformly between near and far bounds derived from sparse 3D reconstruction, this method computes adaptive bounds based on cross-view geometric consistency errors, ensuring samples are concentrated around plausible depths [S1]. It further refines the output depth map using confidence scores derived from RGB reconstruction errors and plane bilateral filtering [S1].

## How it works
**Inputs:**
- $N$ input views with RGB images and camera parameters (intrinsics $K$, poses $T$).
- Adapted monocular depth priors $\{D^i\}_{i=1}^N$ for each view, obtained via scene-specific fine-tuning of a pre-trained monocular depth network [S1].
- Hyperparameters: $K=4$ (number of cross-view errors for error map calculation), $\alpha_l = 0.05$ (lower bound for range), $\alpha_h = 0.15$ (upper bound for range) [S1].

**Outputs:**
- Optimized NeRF parameters $\theta$ representing the implicit radiance field $F_\theta: (x, d) \rightarrow (c, \sigma)$ [S1].
- Refined depth map $D(\mathbf{r})$ for each ray $\mathbf{r}$ [S1].

**Data Structures:**
- **Depth Prior Map:** $D^i \in \mathbb{R}^{H \times W}$ for view $i$.
- **Error Map:** $e_i \in \mathbb{R}^{H \times W}$ for view $i$, storing the average of the $K$ minimum cross-view reprojection errors.
- **Confidence Map:** $S^i \in \mathbb{R}^{H \times W}$ for view $i$, storing per-pixel confidence scores.
- **NeRF Model:** A multi-layer perceptron (MLP) taking position $x$ and direction $d$ as input, outputting color $c$ and density $\sigma$ [S1].

**Steps:**

1.  **Compute Error Maps:**
    For each view $i$, project its depth map $D^i$ to all other views $j \neq i$ using camera intrinsics $K$ and relative pose $T^{i \rightarrow j}$ to obtain projected depth $D^{i \rightarrow j}$ [S1].
    For each pixel $p$ in view $i$, calculate the reprojection error against the depth of view $j$ at the projected location. Collect these errors for all $j \neq i$.
    The error map value $e_i(p)$ for pixel $p$ in view $i$ is the average of the $K$ minimum cross-view depth projection errors [S1].

2.  **Calculate Adaptive Sample Ranges:**
    For each ray $\mathbf{r}$ corresponding to pixel $p$ in view $i$, let $D$ be the adapted depth prior value $D^i(p)$ and $e$ be the error map value $e_i(p)$ [S1].
    Compute the near bound $t_n$ and far bound $t_f$ using clamping:
    $$ t_n = D(1 - \text{clamp}(e, \alpha_l, \alpha_h)) $$
    $$ t_f = D(1 + \text{clamp}(e, \alpha_l, \alpha_h)) $$
    where $\text{clamp}(x, a, b) = \max(a, \min(b, x))$ [S1].
    *Note:* If the depth prior is unreliable (high error), the range expands towards the original NeRF bounds. If the prior is accurate (low error), the range contracts around $D$ [S1].

3.  **Stratified Sampling:**
    Partition the interval $[t_n, t_f]$ into $M$ bins [S1].
    For each bin $i \in [1, M]$, sample a point $t_i$ uniformly from the bin interval:
    $$ t_i \sim \mathcal{U} \left[ t_n + \frac{i-1}{M}(t_f - t_n), \, t_n + \frac{i}{M}(t_f - t_n) \right] $$
    Query the NeRF at positions $t_i \cdot \mathbf{d}$ (where $\mathbf{d}$ is the ray direction) to obtain color $c_i$ and density $\sigma_i$ [S1].

4.  **Volume Rendering:**
    Compute accumulated transmittance $T_i$:
    $$ T_i = \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right) $$
    where $\delta_i = t_{i+1} - t_i$ is the distance between adjacent samples [S1].
    Render the color $C(\mathbf{r})$ and depth $D(\mathbf{r})$:
    $$ C(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
    $$ D(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) t_i $$
    [S1]

5.  **Compute Confidence Scores:**
    For each pixel $j$ in view $i$, compute the confidence score $S^i_j$ based on the $L_1$ difference between the ground truth RGB image $C_{gt}^i(j)$ and the rendered RGB image $C_{render}^i(j)$, normalized by 255 [S1]:
    $$ S^i_j = 1 - \frac{1}{3} \| C_{gt}^i(j) - C_{render}^i(j) \|_1 $$
    The norm is taken over the 3 color channels [S1].

6.  **Depth Refinement (Post-Processing):**
    Apply plane bilateral filtering to the rendered depth map, weighted by the confidence scores $S^i_j$ [S1]. This step improves depth quality in regions where the rendered RGB images are not accurate [S1].

7.  **Optimization Loop:**
    Backpropagate gradients from the photometric loss $\mathcal{L}_{\text{photo}} = \| C_{gt} - C(\mathbf{r}) \|^2$ through the volume rendering process to update NeRF parameters $\theta$ [S1]. The guided sampling ensures that the optimization focuses on geometrically plausible regions, reducing shape-radiance ambiguity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def guided_optimization_neRF(views, adapted_depth_priors, K=4, alpha_l=0.05, alpha_h=0.15, M=48):
    """
    views: List of view objects containing RGB image, camera intrinsics K_cam, pose T.
    adapted_depth_priors: List of depth maps D^i corresponding to each view.
    K: Number of minimum cross-view errors to average for error map.
    alpha_l, alpha_h: Bounds for adaptive range expansion.
    M: Number of samples per ray.
    """
    N = len(views)
    error_maps = []
    
    # Step 1: Compute Error Maps
    for i in range(N):
        H, W = views[i].rgb.shape[:2]
        error_map = np.zeros((H, W))
        depths_for_avg = []
        
        for j in range(N):
            if i == j:
                continue
            
            # Project depth from view i to view j
            # proj(K, T, D) returns projected coordinates and depth values
            # Note: Implementation details of proj depend on specific projection library,
            # but logically it uses K and T^{i->j} to map D^i to view j's coordinate system.
            proj_coords, proj_depth = project_depth(views[i].K, views[i].T, views[j].T, adapted_depth_priors[i])
            
            # Get depth from view j at projected coordinates
            # Handle occlusions/invalid projections by masking or ignoring
            depth_j_at_proj = sample_depth_at_coords(views[j].depth_map, proj_coords)
            
            # Calculate reprojection error
            # Only consider valid pixels where both depths are available
            valid_mask = (proj_depth > 0) & (depth_j_at_proj > 0)
            errors = np.abs(proj_depth[valid_mask] - depth_j_at_proj[valid_mask]) / proj_depth[valid_mask]
            
            # Store errors for this view pair
            # We need to aggregate errors across all j to find the K minimums per pixel
            # This requires a more complex aggregation structure, simplified here conceptually
            depths_for_avg.append(errors)
            
        # Aggregate errors to find K minimums per pixel
        # In practice, this involves collecting all cross-view errors for each pixel
        # and taking the average of the K smallest.
        # For pseudocode clarity, assuming a helper function exists:
        error_map = compute_k_min_average_errors(depths_for_avg, K)
        error_maps.append(error_map)

    # Step 2-7: Training Loop
    nerf_model = initialize_NeRF()
    
    for iteration in range(num_iterations):
        for i in range(N):
            view = views[i]
            depth_prior = adapted_depth_priors[i]
            err_map = error_maps[i]
            
            # Sample rays for this view
            rays = generate_rays(view)
            rendered_colors = []
            rendered_depths = []
            
            for ray in rays:
                pixel_idx = ray.pixel_index
                D = depth_prior[pixel_idx]
                e = err_map[pixel_idx]
                
                # Step 2: Adaptive Ranges
                clamped_e = np.clip(e, alpha_l, alpha_h)
                t_n = D * (1 - clamped_e)
                t_f = D * (1 + clamped_e)
                
                # Step 3: Stratified Sampling
                t_samples = np.linspace(t_n, t_f, M + 1)
                # Add uniform noise within each bin
                noise = np.random.uniform(0, 1, M)
                t_samples = t_n + (t_samples[1:] - t_samples[:-1]) * noise + t_samples[:-1]
                
                # Query NeRF
                positions = ray.origin + t_samples[:, np.newaxis] * ray.direction
                colors, densities = nerf_model(positions, ray.direction)
                
                # Step 4: Volume Rendering
                deltas = t_samples[1:] - t_samples[:-1]
                deltas = np.append(deltas, 1e-10) # Avoid division by zero
                
                alpha = 1 - np.exp(-densities * deltas)
                transmittance = np.cumprod(np.concatenate([[1.0], 1 - alpha[:-1]]))
                
                weights = alpha * transmittance
                C_render = np.sum(weights[:, np.newaxis] * colors, axis=0)
                D_render = np.sum(weights * t_samples, axis=0)
                
                rendered_colors.append(C_render)
                rendered_depths.append(D_render)
                
            # Step 5: Confidence Scores (for post-processing, not used in loss directly in this variant)
            # S^i_j = 1 - 1/3 * L1(C_gt, C_render)
            
            # Optimization
            gt_rgb = view.rgb
            loss = np.mean((np.array(rendered_colors) - gt_rgb) ** 2)
            loss.backward()
            optimizer.step()
            
    # Step 6: Depth Refinement
    final_depth_maps = []
    for i in range(N):
        # Render depth for all pixels
        # ... (rendering logic for full image) ...
        depth_map = render_full_depth(nerf_model, views[i])
        confidence_map = compute_confidence(views[i].rgb, depth_map, nerf_model, views[i])
        refined_depth = plane_bilateral_filter(depth_map, confidence_map)
        final_depth_maps.append(refined_depth)
        
    return nerf_model, final_depth_maps

def project_depth(K, T_src_to_tgt, D_src):
    """
    Projects depth map D_src from source view to target view using K and relative pose.
    Returns projected coordinates and depth values.
    """
    # Implementation depends on specific projection math (e.g., using inverse depth or disparity)
    # This is a placeholder for the geometric projection step described in [S1]
    pass

def compute_k_min_average_errors(errors_list, K):
    """
    Computes the average of the K minimum errors for each pixel across all views.
    """
    # Implementation details omitted for brevity
    pass

def plane_bilateral_filter(depth_map, confidence_map):
    """
    Applies plane bilateral filtering weighted by confidence.
    """
    # Implementation details omitted for brevity
    pass
```

## Subtopics

### ENeRF with Guided Sampling Training
Integrates pixel and depth-guided ray sampling into the Efficient NeRF (ENeRF) framework [S2]. It samples fewer rays per iteration than standard ENeRF, achieving improved performance with lower GPU memory usage [S2]. The loss function combines MSE loss $\mathcal{L}_{mse} = \frac{1}{N_r} \sum_{i=1}^{N_r} \| \hat{C}_i - C_i \|^2$ and perceptual loss $\mathcal{L}_{perc} = \frac{1}{N_i} \sum_{i=1}^{N_i} \| \Phi(\hat{I}_i) - \Phi(I_i) \|$, with total loss $\mathcal{L} = \mathcal{L}_{mse} + \lambda' \mathcal{L}_{perc}$ where $\lambda' = 0.01$ [S2]. It is used for generalization on DTU, fine-tuning on novel scenes, and testing on Real Forward-facing, NeRF Synthetic, ZJU-MoCap, and digital human datasets [S2].

## Operating characteristics
- **Time Complexity:** Depends on the number of samples $M$ per ray and the number of rays sampled per iteration. The guided sampling reduces the effective search space, potentially accelerating convergence in texture-less regions [S1]. ENeRF variant uses fewer rays per iteration [S2].
- **Memory Complexity:** Standard NeRF memory usage is proportional to the number of samples and network size. The ENeRF variant with guided sampling occupies less GPU memory due to fewer rays sampled per iteration [S2].
- **Performance:** Outperforms original NeRF in depth estimation quality on indoor scenes (e.g., ScanNet) by mitigating shape-radiance ambiguity [S1].
- **Strengths:** Effectively handles poorly textured indoor areas (walls) where standard NeRF fails due to shape-radiance ambiguity [S1]. Improves depth accuracy by constraining sampling to plausible ranges [S1].
- **Limitations:** Relies on the quality of adapted monocular depth priors. Applying existing correspondence-based neural optimization (like CVD) to adapted depth priors can degrade quality due to unreliable flow correspondences in poorly textured regions [S1].
- **When to use:** Indoor multi-view stereo and depth estimation where texture is sparse and monocular depth priors are available.
- **When not to use:** Outdoor scenes with rich texture where standard NeRF sampling is sufficient, or when high-quality monocular depth priors are not available.

## Background
Guided Optimization of NeRF was proposed to address the shape-radiance ambiguity inherent in Neural Radiance Fields (NeRF) [S1]. NeRF, initially proposed in [33], represents implicit radiance fields with an MLP [S1]. While NeRF works well for richly textured outdoor scenes, it struggles in poorly textured indoor environments, synthesizing high-quality RGB images with incorrect geometry [S1]. The method integrates adapted monocular depth priors into the NeRF optimization process, using cross-view consistency to guide sampling [S1]. It was evaluated on the ScanNet dataset for multi-view depth estimation [S1].

## Sources
[S1] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
