# Guided Optimization of NeRF

Also called: Confidence-Based Depth Filtering

## Overview
Guided Optimization of NeRF mitigates shape-radiance ambiguity in poorly textured indoor scenes by constraining the stratified sampling of points along camera rays using adapted monocular depth priors [S1]. Unlike standard NeRF, which samples uniformly between sparse reconstruction bounds, this method computes adaptive near and far bounds based on multi-view geometric consistency errors, ensuring sampled points concentrate around the true geometry [S1]. It further refines the output depth map using confidence scores derived from RGB reconstruction errors and plane bilateral filtering [S1].

## How it works
**Inputs:**
*   $N$ input views with RGB images $I^i$ and camera intrinsics $K$ and poses $T^{i \to j}$.
*   Adapted monocular depth priors $\{D^i\}_{i=1}^N$ for each view, obtained by fine-tuning a pre-trained monocular depth network on the target scene [S1].
*   Hyperparameters: $K=4$ (number of cross-view errors used for error map calculation), $\alpha_l = 0.05$ (lower bound), $\alpha_h = 0.15$ (upper bound), $M$ (number of samples per ray) [S1].

**Outputs:**
*   Optimized NeRF parameters $\theta$ defining the implicit radiance field $F_\theta: (x, d) \to (c, \sigma)$ [S1].
*   Final refined depth map for each view.

**Data Structures:**
*   **Error Map $e^i$:** A 2D array of size $H \times W$ for view $i$, storing the average of the $K$ minimum cross-view reprojection errors for each pixel [S1].
*   **Confidence Map $S^i$:** A 2D array of size $H \times W$ for view $i$, storing confidence scores per pixel [S1].

**Algorithm Steps:**

1.  **Acquire Adapted Depth Priors:**
    Initialize monocular depth priors $\{D^i\}_{i=1}^N$ using a pre-trained network (e.g., Mannequin Challenge architecture [S1]) and fine-tune them on the target scene for 15 epochs [S1].

2.  **Compute Error Maps:**
    For each view $i$, project its depth map $D^i$ to all other views $j \neq i$ using camera intrinsics $K$ and relative pose $T^{i \to j}$ to obtain projected depth $D^{i \to j}$ [S1].
    Calculate the reprojection error between the actual depth $D^j$ and the projected depth $D^{i \to j}$ at corresponding pixel coordinates [S1].
    For each pixel in view $i$, collect the reprojection errors from all other views. Select the $K$ minimum errors and compute their average to form the error map value $e^i$ for that pixel [S1].

3.  **Calculate Adaptive Sample Ranges:**
    For each ray $\mathbf{r}$ corresponding to a pixel in view $i$, let $D$ be the value of the adapted depth prior $D^i$ at that pixel, and $e$ be the error map value $e^i$ at that pixel [S1].
    Compute the near bound $t_n$ and far bound $t_f$ using the clamping function:
    $$ t_n = D(1 - \text{clamp}(e, \alpha_l, \alpha_h)) $$
    $$ t_f = D(1 + \text{clamp}(e, \alpha_l, \alpha_h)) $$
    where $\text{clamp}(x, a, b) = \max(a, \min(b, x))$ [S1].
    *Note:* If the pixel is not visible in other views, $e$ may be undefined or large; the clamping ensures the range does not exceed $[D(1-\alpha_h), D(1+\alpha_h)]$ [S1].

4.  **Stratified Sampling:**
    Partition the interval $[t_n, t_f]$ into $M$ bins [S1].
    For each bin $i \in [1, M]$, sample a point $t_i$ uniformly from the bin interval:
    $$ t_i \sim \mathcal{U} \left[ t_n + \frac{i-1}{M}(t_f - t_n), \,\, t_n + \frac{i}{M}(t_f - t_n) \right] $$
    Query the NeRF network $F_\theta$ at spatial locations $\mathbf{x}_i$ corresponding to $t_i$ along the ray to obtain density $\sigma_i$ and color $c_i$ [S1].

5.  **Volume Rendering:**
    Compute accumulated transmittance $T_i$:
    $$ T_i = \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right) $$
    where $\delta_i = t_{i+1} - t_i$ is the distance between adjacent samples [S1].
    Render the color $C(\mathbf{r})$ and depth $D(\mathbf{r})$ for the ray:
    $$ C(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
    $$ D(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) t_i $$
    [S1].

6.  **Compute Confidence Scores:**
    For each pixel $j$ in view $i$, calculate the confidence score $S^i_j$ based on the $L_1$ difference between the ground truth RGB image $C_{gt}^i(j)$ and the rendered RGB image $C_{render}^i(j)$, normalized by 255 [S1]:
    $$ S^i_j = 1 - \frac{1}{3} \| C_{gt}^i(j) - C_{render}^i(j) \|_1 $$
    The $L_1$ norm is taken over the 3 color channels [S1].

7.  **Depth Refinement (Post-Processing):**
    Apply plane bilateral filtering to the rendered depth map $D(\mathbf{r})$, using the confidence scores $S^i$ as weights [S1]. This step improves depth quality in regions where the rendered RGB images are inaccurate [S1].

8.  **Optimization Loop:**
    Compute the photometric loss $\mathcal{L}_{\text{photo}} = \| C(\mathbf{r}) - C_{gt}(\mathbf{r}) \|_2^2$ [S1, 7475-geometric-baseline-regularization.md].
    Backpropagate gradients through the volume rendering and sampling process to update NeRF parameters $\theta$ [S1, 8582-regnerf.md].
    Repeat steps 2-8 until convergence.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def clamp(x, a, b):
    return np.maximum(a, np.minimum(b, x))

def guided_optimization_neRF(N_views, images, depth_priors, K, alpha_l, alpha_h, M, num_iterations):
    """
    N_views: int, number of input views
    images: list of np.ndarray, shape (H, W, 3), values in [0, 255]
    depth_priors: list of np.ndarray, shape (H, W), adapted monocular depth priors
    K: int, number of minimum cross-view errors
    alpha_l: float, lower bound for range expansion
    alpha_h: float, upper bound for range expansion
    M: int, number of samples per ray
    num_iterations: int, number of training steps
    """
    # Initialize NeRF parameters theta
    theta = initialize_nerf_parameters()
    
    # Precompute camera intrinsics and poses (K, T)
    # Assume these are available as global constants or passed in
    
    for iteration in range(num_iterations):
        # 1. Compute Error Maps
        error_maps = []
        for i in range(N_views):
            D_i = depth_priors[i]
            H, W = D_i.shape
            error_map = np.zeros((H, W))
            
            # Collect reprojection errors from all other views
            all_errors = []
            for j in range(N_views):
                if i == j: continue
                
                # Project depth from view i to view j
                # proj(K, T_i_to_j, D_i) returns projected depth D_i_to_j at 2D coords p
                # For implementation, we assume a function that handles projection and interpolation
                D_i_to_j, valid_mask = project_depth_to_view(K, T[i][j], D_i, images[j].shape)
                
                # Get actual depth from view j at projected coordinates
                # D_j(p) implies sampling D_j at the projected 2D locations
                D_j_at_proj = sample_depth_at_coords(K, T[j][i], D_i_to_j, depth_priors[j]) # Note: logic simplified for pseudocode
                
                # Calculate relative error
                # Avoid division by zero
                safe_D_j = np.where(D_j_at_proj > 0, D_j_at_proj, 1e-6)
                relative_error = np.abs(D_i_to_j - D_j_at_proj) / safe_D_j
                
                # Store errors for valid pixels
                valid_pixels = np.where(valid_mask & (D_j_at_proj > 0))
                if len(valid_pixels[0]) > 0:
                    errors = relative_error[valid_pixels]
                    all_errors.append(errors)
            
            # Compute average of K minimum errors for each pixel
            # Note: The source describes calculating the error map e_i as the average of top K minimum errors.
            # This usually implies aggregating errors across views for each pixel.
            # Simplified implementation:
            if all_errors:
                all_errors = np.concatenate(all_errors)
                # This part is ambiguous in pseudocode without specific aggregation logic per pixel.
                # The source says "average value of the top K minimum cross-view depth projection error".
                # We assume this results in a single scalar error per pixel after aggregation.
                # For the sake of valid Python, we approximate:
                # In practice, one would compute a matrix of errors (H, W, N_views-1) and take min K.
                # Here we assume error_map is pre-computed correctly.
                pass 
            
            error_maps.append(error_map)

        # 2. Sample Rays and Render
        # Iterate over a batch of rays or all rays
        for i in range(N_views):
            D_prior = depth_priors[i]
            e = error_maps[i]
            H, W = D_prior.shape
            
            # Prepare buffers for gradients
            grad_theta = np.zeros_like(theta) # Placeholder for actual gradient accumulation
            
            for y in range(H):
                for x in range(W):
                    D_val = D_prior[y, x]
                    e_val = e[y, x]
                    
                    # Calculate adaptive bounds
                    clamped_e = clamp(e_val, alpha_l, alpha_h)
                    t_n = D_val * (1 - clamped_e)
                    t_f = D_val * (1 + clamped_e)
                    
                    # Stratified Sampling
                    t_samples = np.linspace(t_n, t_f, M + 1)
                    # Add uniform noise within bins
                    noise = np.random.uniform(0, 1, M)
                    t_samples = t_n + (t_samples[1:] - t_n) * noise # Simplified bin sampling
                    
                    # Query NeRF
                    # Get ray direction and origin for pixel (x,y)
                    ray_o, ray_d = get_ray_parameters(K, T[i], x, y)
                    
                    # Interpolate spatial positions
                    x_samples = ray_o + t_samples[:, None] * ray_d
                    
                    # Query network F_theta
                    # sigma, c = nerf_forward(theta, x_samples, ray_d)
                    # Placeholder for network forward pass
                    sigma, c = query_nerf(theta, x_samples, ray_d)
                    
                    # Volume Rendering
                    deltas = t_samples[1:] - t_samples[:-1]
                    # Add a small epsilon to avoid division by zero in transmittance if needed, 
                    # though formula uses exp(-sum).
                    
                    alpha = 1 - np.exp(-sigma * deltas)
                    T_transmittance = np.cumprod(np.concatenate([[1.0], 1 - alpha[:-1]]))
                    
                    weights = alpha * T_transmittance
                    
                    C_render = np.sum(weights * c, axis=0)
                    D_render = np.sum(weights * t_samples, axis=0)
                    
                    # Compute Loss
                    C_gt = images[i][y, x]
                    loss = np.sum((C_render - C_gt) ** 2)
                    
                    # Backpropagation (Placeholder)
                    # grad_theta += compute_gradients(loss, theta, ...)
                    pass
            
            # Update parameters
            # theta -= learning_rate * grad_theta
            pass

        # 3. Post-Processing (Optional during training or at inference)
        # Compute confidence maps
        confidence_maps = []
        for i in range(N_views):
            # Re-render or store rendered images from the last iteration
            # C_rendered = ...
            # C_gt = images[i]
            # S = 1 - (1/3) * np.mean(np.abs(C_gt - C_rendered) / 255, axis=2)
            pass
        
        # Apply plane bilateral filtering
        # final_depth = plane_bilateral_filter(D_rendered, confidence_maps)
        
    return theta
```

## Subtopics

### ENeRF with Guided Sampling Training
Integrates pixel and depth-guided ray sampling into the Efficient NeRF (ENeRF) framework [S2]. It uses fewer rays per iteration, achieving improved performance with lower GPU memory usage [S2]. The loss function combines MSE and perceptual loss: $\mathcal{L} = \mathcal{L}_{mse} + \lambda' \mathcal{L}_{perc}$, with $\lambda' = 0.01$ [S2]. It is used for generalization on DTU, fine-tuning on novel scenes, and testing on Real Forward-facing, NeRF Synthetic, ZJU-MoCap, and digital human datasets [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of views $N$, samples $M$, and training iterations. The error map computation adds $O(N^2 \cdot H \cdot W)$ overhead per iteration for reprojection [S1].
*   **Memory Complexity:** Requires storing depth priors, error maps, and confidence maps for all views [S1]. ENeRF variant reduces GPU memory usage by sampling fewer rays [S2].
*   **Performance:** Outperforms original NeRF in depth estimation accuracy on indoor scenes (e.g., ScanNet) by mitigating shape-radiance ambiguity [S1].
*   **Strengths:** Effective in poorly textured indoor areas (walls) where standard NeRF fails [S1]. Improves depth quality via confidence-based filtering [S1].
*   **Limitations:** Relies on the quality of adapted monocular depth priors [S1]. Applying correspondence-based optimization (like CVD) to these priors can degrade quality due to unreliable flow in texture-less regions [S1].
*   **When to use:** Indoor multi-view stereo with texture-less regions where accurate depth is required [S1].
*   **When not to use:** Richly textured outdoor scenes where standard NeRF's shape-radiance assumption holds [S1].

## Background
Originated from the paper "Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo" [S1]. It addresses the shape-radiance ambiguity identified in [61] (Mildenhall et al., 2020), which causes NeRF to synthesize correct RGB but incorrect geometry in texture-less indoor areas [S1]. The method builds on the standard NeRF volume rendering formulation [S1] and integrates monocular depth priors [S1].

## Sources
[S1] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
