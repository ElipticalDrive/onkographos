# Guided Optimization of NeRF

Also called: Depth-Guided NeRF Optimization, Guided Sampling for NeRF

## Overview
Guided Optimization of NeRF mitigates the shape-radiance ambiguity inherent in Neural Radiance Fields (NeRF) for indoor scenes by restricting the sampling of points along camera rays to ranges defined by adapted monocular depth priors [S1]. Unlike standard NeRF, which samples uniformly between sparse reconstruction bounds and often fails in texture-less regions, this method uses multi-view geometric consistency to compute adaptive near and far bounds, ensuring sampled points concentrate around plausible geometry [S1].

## How it works
**Inputs:** $N$ input RGB images with known camera intrinsics $K$ and poses; a pre-trained monocular depth network (e.g., Mannequin Challenge architecture [S1]); ground truth RGB images for supervision.
**Outputs:** Optimized NeRF parameters $\theta$ (defining $F_\theta: (x, d) \rightarrow (c, \sigma)$); final depth map.

**Data Structures:**
*   **Adapted Depth Priors:** $\{D^i\}_{i=1}^N$, where $D^i$ is the depth map for view $i$.
*   **Error Maps:** $\{e_i\}_{i=1}^N$, scalar maps where $e_i$ represents the reprojection error for pixel $i$.
*   **Sample Ranges:** For each ray $\mathbf{r}$, a pair $[t_n, t_f]$ defining the near and far clipping distances.
*   **Confidence Map:** $\{S^i_j\}$, scalar values per pixel indicating reconstruction quality.

**Numbered Steps:**

1.  **Acquire Adapted Depth Priors:**
    Initialize monocular depth priors $\{D^i\}_{i=1}^N$ for the $N$ input views. These are typically obtained by fine-tuning a pre-trained monocular depth network on the specific target scene for a fixed number of epochs (e.g., 15 epochs) [S1].

2.  **Compute Error Maps via Cross-View Projection:**
    For each view $i$, project its depth map $D^i$ to all other views $j \neq i$ to compute geometric consistency errors.
    a. Project depth from view $i$ to view $j$:
       $$p^{i \rightarrow j}, D^{i \rightarrow j} = \text{proj}(K, T^{i \rightarrow j}, D^i)$$
       where $T^{i \rightarrow j}$ is the relative pose transformation, $p^{i \rightarrow j}$ are the 2D coordinates in view $j$, and $D^{i \rightarrow j}$ are the projected depth values [S1].
    b. Sample the depth from the target view $j$ at these coordinates:
       $$D^{j'} = D^j(p^{i \rightarrow j})$$
    c. Calculate the relative reprojection error between $D^{j'}$ and $D^{i \rightarrow j}$.
    d. For each pixel in view $i$, the error map value $e_i$ is the average of the top $K$ minimum cross-view depth projection errors. The parameter $K=4$ is used [S1].

3.  **Calculate Adaptive Sample Ranges:**
    For each ray $\mathbf{r}$ corresponding to a pixel, determine the near ($t_n$) and far ($t_f$) bounds using the adapted depth prior $D$ and the error map value $e$ for that pixel:
    $$t_n = D(1 - \text{clamp}(e, \alpha_l, \alpha_h))$$
    $$t_f = D(1 + \text{clamp}(e, \alpha_l, \alpha_h))$$
    Here, $\alpha_l = 0.05$ and $\alpha_h = 0.15$ are the lower and upper bounds for the relative range expansion [S1]. The function $\text{clamp}(x, a, b)$ restricts $x$ to the interval $[a, b]$. Pixels with low error (high confidence) have narrow ranges centered on $D$; pixels with high error have wider ranges, approaching standard NeRF sampling [S1].

4.  **Stratified Sampling:**
    Partition the interval $[t_n, t_f]$ into $M$ bins. For each bin $i \in [1, M]$, sample a point $t_i$ uniformly from the bin interval:
    $$t_i \sim \mathcal{U} \left[ t_n + \frac{i-1}{M}(t_f - t_n), \,\, t_n + \frac{i}{M}(t_f - t_n) \right]$$
    This ensures points are distributed along the ray segment defined by the depth-guided bounds [S1].

5.  **NeRF Forward Pass:**
    Query the neural radiance field $F_\theta$ at each sample point $t_i$ (and its direction) to obtain density $\sigma_i$ and color $c_i$. Compute the distance between adjacent samples $\delta_i = t_{i+1} - t_i$ and the accumulated transmittance $T_i = \exp \left(-\sum_{j=1}^{i-1} \sigma_j \delta_j \right)$ [S1].

6.  **Volume Rendering:**
    Render the predicted RGB color $C(\mathbf{r})$ and depth $D(\mathbf{r})$ for the ray:
    $$C(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) c_i$$
    $$D(\mathbf{r}) = \sum_{i=1}^{M} T_i (1 - e^{-\sigma_i \delta_i}) t_i$$
    where $C(\mathbf{r})$ is the rendered color, $D(\mathbf{r})$ is the rendered depth, $c_i$ is the color at sample $i$, and $t_i$ is the depth at sample $i$ [S1].

7.  **Compute Confidence Scores:**
    Calculate a confidence score $S^i_j$ for each pixel $j$ in view $i$ based on the $L_1$ reconstruction error between the ground truth RGB $C_{gt}^i(j)$ and the rendered RGB $C_{render}^i(j)$. Values are normalized by dividing by 255:
    $$S^i_j = 1 - \frac{1}{3} \| C_{gt}^i(j) - C_{render}^i(j) \|_1$$
    This score reflects the reliability of the depth prediction for that pixel [S1].

8.  **Post-Processing (Optional):**
    Apply plane bilateral filtering to the final depth map to refine depth quality, particularly in regions where the rendered RGB images were inaccurate [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def guided_optimization_neRF(N, images, K, poses, depth_net, M=64, K_proj=4, alpha_l=0.05, alpha_h=0.15):
    """
    N: Number of input views
    images: List of N RGB images (H, W, 3), values 0-255
    K: Camera intrinsics matrix
    poses: List of N pose matrices (4x4)
    depth_net: Function to predict depth from RGB
    M: Number of samples per ray
    K_proj: Number of min errors to average for error map
    alpha_l, alpha_h: Bounds for sample range expansion
    """
    
    # Step 1: Acquire adapted depth priors
    # Fine-tune depth_net on images for 15 epochs to get scene-specific priors
    # D_priors is a list of N depth maps (H, W)
    D_priors = fine_tune_monocular_depth(depth_net, images, epochs=15)
    
    # Step 2: Compute Error Maps
    error_maps = []
    for i in range(N):
        D_i = D_priors[i]
        min_errors = []
        
        # Project D_i to all other views
        for j in range(N):
            if i == j: continue
            
            # Get relative pose T_i_to_j
            T_ij = poses[j] @ np.linalg.inv(poses[i])
            
            # Project depth D_i to view j
            # proj function handles K and T_ij
            # p_ij: (H, W, 2) coordinates in view j
            # D_proj_ij: (H, W) projected depth values
            p_ij, D_proj_ij = project_depth(D_i, K, T_ij)
            
            # Sample depth from view j's prior at projected coordinates
            # D_j_prime: (H, W) depth values from D_priors[j]
            D_j_prime = sample_at_coords(D_priors[j], p_ij)
            
            # Calculate relative error
            # Avoid division by zero
            valid_mask = D_proj_ij > 0
            relative_error = np.abs(D_j_prime - D_proj_ij) / (D_proj_ij + 1e-6)
            
            # Flatten valid errors
            flat_errors = relative_error[valid_mask]
            min_errors.append(flat_errors)
        
        # Average of top K_proj minimum errors
        # Combine all errors from other views
        all_errors = np.concatenate(min_errors)
        # Sort and take top K_proj smallest
        all_errors_sorted = np.sort(all_errors)
        avg_min_error = np.mean(all_errors_sorted[:K_proj])
        
        # Create error map (broadcast avg_min_error to all pixels, or compute per-pixel if logic allows)
        # Note: The source says "average value of the top K minimum cross-view depth projection error"
        # It implies a global or local statistic. Given "error map value e_i", it suggests per-pixel or per-view.
        # The formula uses e for a pixel. The text says "calculate the error map... e_i as the average value...".
        # Usually, this is computed per pixel by averaging the min errors across views for that pixel.
        # Re-reading: "calculate the depth reprojection error... define the error map... e_i as the average value of the top K minimum..."
        # This phrasing is slightly ambiguous. However, standard practice and the formula usage (e is per pixel) 
        # suggests computing a per-pixel error map.
        # Let's assume per-pixel calculation for correctness of implementation.
        
        pixel_error_map = np.zeros_like(D_i)
        for h in range(H):
            for w in range(W):
                if not valid_mask[h,w]:
                    pixel_error_map[h,w] = 1.0 # Default high error if invalid
                    continue
                
                # Collect errors for this pixel from all other views
                pixel_errors = []
                for j in range(N):
                    if i == j: continue
                    T_ij = poses[j] @ np.linalg.inv(poses[i])
                    p_ij, D_proj_ij = project_depth(D_i, K, T_ij)
                    D_j_prime = sample_at_coords(D_priors[j], p_ij)
                    
                    # Get specific pixel error
                    # Note: p_ij is (H,W,2). We need to index carefully.
                    # Assuming p_ij[h,w] gives the coordinate.
                    coord = p_ij[h,w]
                    # Sample D_j_prime at coord
                    val_j = sample_at_coords_single(D_priors[j], coord)
                    val_proj = D_proj_ij[h,w]
                    
                    if val_proj > 0:
                        err = np.abs(val_j - val_proj) / (val_proj + 1e-6)
                        pixel_errors.append(err)
                
                if len(pixel_errors) > 0:
                    pixel_errors.sort()
                    # Average of top K_proj
                    k_actual = min(K_proj, len(pixel_errors))
                    pixel_error_map[h,w] = np.mean(pixel_errors[:k_actual])
                else:
                    pixel_error_map[h,w] = 1.0
        
        error_maps.append(pixel_error_map)

    # Step 3 & 4: Sampling and Rendering Loop (Optimization)
    # Initialize NeRF parameters theta
    theta = init_neRF_params()
    
    for iteration in range(max_iterations):
        # Pick a random view i and pixel j
        i = random.randint(0, N-1)
        j = random.randint(0, H-1)
        k = random.randint(0, W-1)
        
        # Get ray r for pixel (j,k) in view i
        r = get_ray(K, poses[i], j, k)
        
        # Get adapted depth D and error e for this pixel
        D = D_priors[i][j, k]
        e = error_maps[i][j, k]
        
        # Clamp error
        e_clamped = np.clip(e, alpha_l, alpha_h)
        
        # Calculate bounds
        t_n = D * (1 - e_clamped)
        t_f = D * (1 + e_clamped)
        
        # Stratified Sampling
        samples_t = []
        for m in range(M):
            t_min = t_n + (m / M) * (t_f - t_n)
            t_max = t_n + ((m + 1) / M) * (t_f - t_n)
            t_sample = np.random.uniform(t_min, t_max)
            samples_t.append(t_sample)
        
        # Query NeRF
        # samples_x, samples_d derived from r and samples_t
        sigmas, colors = query_neRF(theta, samples_t, r)
        
        # Volume Rendering
        deltas = np.diff(samples_t + [samples_t[-1]]) # Approximation for last delta
        # T_i calculation
        alpha = 1 - np.exp(-sigmas * deltas)
        T = np.cumprod(np.exp(-np.concatenate([[0], sigmas[:-1] * deltas[:-1]]))) # Accumulated transmittance
        
        # Rendered Color
        weights = alpha * T
        C_render = np.sum(weights * colors, axis=0)
        
        # Rendered Depth
        D_render = np.sum(weights * samples_t, axis=0)
        
        # Loss Computation
        C_gt = images[i][j, k] / 255.0
        loss_rgb = np.mean(np.abs(C_gt - C_render))
        
        # Update theta using gradient descent
        update_params(theta, loss_rgb)
        
    # Step 8: Post-processing
    final_depth = apply_plane_bilateral_filter(D_render_all)
    
    return theta, final_depth

def clamp(x, a, b):
    return max(a, min(b, x))

def project_depth(D, K, T):
    # Project depth map D using K and pose T to new view
    # Returns 2D coordinates and projected depth values
    pass

def sample_at_coords(D, coords):
    # Sample depth map D at given 2D coordinates
    pass

def sample_at_coords_single(D, coord):
    # Sample depth map D at single 2D coordinate
    pass

def get_ray(K, pose, u, v):
    # Generate camera ray from pixel coordinates
    pass

def query_neRF(theta, ts, ray):
    # Query NeRF network for sigma and color at ts along ray
    pass
```

## Subtopics

### ENeRF with Guided Sampling Training
Integrates pixel and depth-guided ray sampling into the Efficient NeRF (ENeRF) framework [S2]. It samples fewer rays per iteration than standard ENeRF, achieving improved performance with lower GPU memory usage [S2]. It uses a total loss $\mathcal{L} = \mathcal{L}_{mse} + \lambda' \mathcal{L}_{perc}$ with $\lambda' = 0.01$ [S2]. It is used for rapid generalization on the DTU dataset and fine-tuning on novel scenes, including digital humans [S2].

## Operating characteristics
**Time and Memory Complexity:** The method requires additional computation for multi-view depth projection and error map calculation during initialization [S1]. During training, the adaptive sampling range calculation is lightweight [S1]. The variant ENeRF with guided sampling reduces the number of rays sampled per iteration, leading to lower GPU memory usage compared to standard ENeRF [S2].

**Measured Performance:** On the ScanNet dataset, the method outperforms state-of-the-art depth estimation methods (including DeepV2D, DELTAS, and Atlas) in all metrics, despite not using ground-truth depth supervision for training [S1]. It effectively mitigates shape-radiance ambiguity in indoor scenes [S1].

**Strengths and Limitations:**
*   **Strengths:** Successfully addresses shape-radiance ambiguity in poorly textured indoor areas (e.g., walls) where standard NeRF fails [S1]. Provides accurate depth estimation by explicitly limiting sampling ranges around plausible geometry [S1].
*   **Limitations:** Relies on the quality of adapted monocular depth priors. Applying existing correspondence-based neural optimization (like CVD) to these priors can degrade quality due to unreliable flow correspondences in texture-less regions [S1].

**When to Use:** Use for indoor multi-view stereo and depth estimation tasks where texture is sparse and standard NeRF produces incorrect geometry despite good RGB reconstruction [S1].

**When Not to Use:** Avoid if high-quality correspondence-based optimization is required, as it may degrade the adapted priors [S1]. Not explicitly designed for dynamic scenes or unknown camera parameters without modification [S1].

## Background
The method was proposed in "Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo" [S1]. It addresses the limitation of original NeRF [33], which assumes correct geometry is favored by smooth surface light fields but fails in texture-less indoor environments [S1]. The approach builds on the Mannequin Challenge monocular depth network [23] and integrates it with NeRF optimization [S1].

## Sources
[S1] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
