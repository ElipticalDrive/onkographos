# Advance Floater Pruning

Also called: Advanced Floater Pruning, Advanced Floater Removal, Floater Pruning, Pruning, Pruning heuristic

## Overview
Advance Floater Pruning is an explicit and adaptive operator designed to identify and eliminate undesired floating artifacts (misplaced Gaussians) in 3D Gaussian Splatting (3DGS) representations, particularly in sparse-view synthesis settings [S1]. It distinguishes itself from standard pruning by analyzing the relative difference between mode-selected depth and alpha-blended depth to detect "floaters"—low-opacity Gaussians positioned close to the camera center that cause misalignment between depth metrics [S1]. Unlike [Depth-Based Gaussian Pruning](12338-depth-based-gaussian-pruning.md), which adjusts thresholds based on scene depth to preserve far-plane points, this method targets geometric inconsistencies caused by transmittance budget depletion near the camera [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained 3DGS representation $G$, a set of training cameras $I$, and curve parameters $a$ and $b$.
*   **Output:** The 3DGS representation $G$ with identified floater Gaussians removed.

**Data Structures:**
*   **3DGS Representation $G$:** Contains Gaussian primitives with attributes including position, covariance, opacity, and color.
*   **Depth Maps:** Rendered per training view $i$:
    *   $d^{alpha}_i$: Alpha-blending depth map.
    *   $d^{mode}_i$: Mode-selection depth map.
*   **Relative Difference Map $\Delta_i$:** Per-pixel relative difference between $d^{mode}_i$ and $d^{alpha}_i$.
*   **Floater Mask $F_i$:** Binary mask indicating pixels with relative difference exceeding the threshold.

**Definitions:**
*   **Alpha-blending depth ($d^{alpha}$):** Calculated as the weighted sum of depths along a ray, influenced by all Gaussians [S1]:
    $$ d^{alpha}_{x,y} = \sum_{i=1}^{N} T_i \alpha_i d_i $$
    where $T_i$ is the accumulated transmittance for the $i$-th Gaussian, $\alpha_i$ is the alpha-compositing weight, and $d_i$ is the depth of the Gaussian [S1].
*   **Mode-selection depth ($d^{mode}$):** The depth value corresponding to the Gaussian with the maximum alpha-compositing weight (or softmax-weighted depth) along the ray [S1].
*   **Relative Difference ($\Delta_i$):** For each training view $i$, the per-pixel relative difference is computed as:
    $$ \Delta_i = \frac{d^{mode}_i - d^{alpha}_i}{d^{alpha}_i} $$
*   **Dip Statistic:** A measure of uni-modality applied to the distribution of $\Delta_i$ values [S1].

**Parameters:**
*   $a = 0.97$: Exponential curve parameter for threshold conversion [S1].
*   $b = -7.5$: Exponential curve parameter for threshold conversion [S1].

**Algorithm Steps:**
1.  Initialize the average dip statistic $\bar{D} = 0.0$.
2.  For each training view $i \in I$:
    a. Render the alpha-blending depth map $d^{alpha}_i$ from $G$ using view $i$.
    b. Render the mode-selection depth map $d^{mode}_i$ from $G$ using view $i$.
    c. Compute the relative difference map $\Delta_i = (d^{mode}_i - d^{alpha}_i) / d^{alpha}_i$.
    d. Compute the dip statistic for the distribution of $\Delta_i$ values.
    e. Add this dip statistic to $\bar{D}$.
3.  Compute the average dip statistic: $\bar{D} = \bar{D} / |I|$, where $|I|$ is the number of training views. This score represents the scene-wide presence of floaters [S1].
4.  For each training view $i \in I$:
    a. Calculate the threshold $\tau_i$ using the percentile function with an exponential curve based on the average dip statistic:
        $$ \tau_i = \text{percentile}(\Delta_i, a e^{b \bar{D}}) $$
        The percentile function selects the value below which a given percentage of $\Delta_i$ values fall. The argument $a e^{b \bar{D}}$ maps the dip statistic to a percentile value [S1].
    b. Generate the floater mask $F_i$ where $F_i = 1$ if $\Delta_i > \tau_i$, and $0$ otherwise.
    c. Identify the indices of all Gaussians contributing to pixels where $F_i = 1$. This includes the mode Gaussian and other Gaussians along those rays [S1].
    d. Remove these identified Gaussians from $G$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def advance_floater_pruning(G, training_cameras, a=0.97, b=-7.5):
    """
    Prunes floaters from 3D Gaussian Splatting representation G.
    
    Args:
        G: 3DGS representation containing Gaussians.
        training_cameras: List of camera parameters for training views.
        a: Curve parameter for threshold conversion.
        b: Curve parameter for threshold conversion.
        
    Returns:
        G: Pruned 3DGS representation.
    """
    total_dip_statistic = 0.0
    num_views = len(training_cameras)
    
    # Step 1: Compute average dip statistic across all views
    for i in range(num_views):
        cam = training_cameras[i]
        
        # Render depth maps
        # Note: Implementation of alpha_blending_depth and mode_selection_depth
        # depends on the specific 3DGS rasterizer.
        d_alpha = alpha_blending_depth(G, cam)
        d_mode = mode_selection_depth(G, cam)
        
        # Compute relative difference
        # Avoid division by zero; handle background pixels if necessary
        # Assuming d_alpha is valid for foreground pixels
        delta_i = (d_mode - d_alpha) / d_alpha
        
        # Compute dip statistic for the distribution of delta_i
        # The dip statistic measures the maximum difference between the 
        # empirical distribution function and the best-fitting unimodal distribution.
        dip_stat = compute_dip_statistic(delta_i.flatten())
        
        total_dip_statistic += dip_stat
        
    average_dip_statistic = total_dip_statistic / num_views
    
    # Step 2: Apply pruning for each view
    for i in range(num_views):
        cam = training_cameras[i]
        
        # Re-render or use cached depth maps
        d_alpha = alpha_blending_depth(G, cam)
        d_mode = mode_selection_depth(G, cam)
        
        delta_i = (d_mode - d_alpha) / d_alpha
        
        # Calculate threshold
        # The percentile argument is a value between 0 and 100.
        # a * exp(b * D_bar) maps the dip statistic to this percentile.
        percentile_value = a * math.exp(b * average_dip_statistic)
        
        # Ensure percentile_value is within valid range [0, 100]
        percentile_value = max(0.0, min(100.0, percentile_value))
        
        threshold = percentile(delta_i.flatten(), percentile_value)
        
        # Generate floater mask
        floater_mask = delta_i > threshold
        
        # Identify Gaussians to remove
        # This requires mapping pixel coordinates in floater_mask back to 
        # the Gaussians that contributed to those pixels during rendering.
        gaussians_to_remove = mask_to_gaussian_indices(G, cam, floater_mask)
        
        # Remove Gaussians
        G.remove_gaussians(gaussians_to_remove)
        
    return G
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $d^{alpha}_i$ is zero (e.g., background pixels), the relative difference $\Delta_i$ is undefined. Implementation must handle background pixels, typically by excluding them from the dip statistic calculation and mask generation [S1].
*   **Transmittance Threshold:** Floaters arise because 3DGS halts rasterization once a transmittance threshold is reached. Near-camera floaters deplete this budget early, preventing convergence at the correct depth [S1].
*   **Bimodal Distribution:** The method relies on the observation that floaters cause a bimodal distribution in $\Delta_i$. If the scene does not exhibit this characteristic (e.g., no floaters), the dip statistic will be low, resulting in a high percentile threshold and minimal pruning [S1].
*   **Timing:** Pruning is applied once after training completes, not during training iterations [S1].

## Operating characteristics
*   **Time Complexity:** The method requires rendering depth maps for all training views twice (once for dip statistic calculation, once for pruning) and computing dip statistics. The complexity is linear with respect to the number of training views and pixels.
*   **Memory Complexity:** Requires storing depth maps and relative difference maps for all training views.
*   **Performance:** Effectively removes "background collapse" and floating artifacts in sparse-input settings [S1].
*   **Strengths:** Explicitly targets geometric inconsistencies (floaters) that are not addressed by opacity-based pruning. Adaptive thresholding based on scene-wide dip statistic allows it to handle varying levels of floater presence [S1].
*   **Limitations:** Only applicable after training. Relies on the assumption that floaters cause bimodal depth difference distributions.

## Background
*   **Origin:** Introduced in the paper "SparseGS: Sparse View Synthesis Using 3D Gaussian Splatting" [S1].
*   **Context:** Developed to address artifacts in 3D Gaussian Splatting when trained on sparse input views. The method complements other regularization techniques like Unseen Viewpoint Regularization and Score Distillation Sampling [S1].
*   **History:** The concept of "floaters" in 3DGS refers to low-opacity Gaussians near the camera that cause misalignment between mode-selection and alpha-blending depths. This pruning heuristic was empirically derived by examining $\Delta_i$ distributions across various scenes [S1].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
