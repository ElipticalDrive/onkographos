# Advance Floater Pruning

Also called: Advanced Floater Pruning, Advanced Floater Removal, Floater Pruning, Pruning heuristic

## Overview
Advance Floater Pruning is an explicit and adaptive operator designed to identify and eliminate undesired floating artifacts (misplaced Gaussians) in 3D Gaussian Splatting (3DGS) representations, particularly in sparse-view synthesis settings [S1]. It distinguishes itself from standard pruning by analyzing the per-pixel relative difference between mode-selected depth and alpha-blended depth to detect "floaters"—low-opacity Gaussians positioned close to the camera center that cause misalignment between depth metrics [S1]. Unlike [Depth-Based Gaussian Pruning](12338-depth-based-gaussian-pruning.md), which adjusts thresholds based on relative scene depth to preserve far-plane points, this method targets geometric inconsistencies caused by near-camera floaters depleting the transmittance budget early [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained 3DGS representation $G$, a set of training cameras $I$, and curve parameters $a$ and $b$ [S1].
*   **Output:** The 3DGS representation $G$ with identified floater Gaussians removed [S1].

**Data Structures:**
*   **3DGS Representation $G$:** Contains Gaussian primitives with attributes including position, covariance, opacity, and color [G].
*   **Depth Maps:** Rendered per training view $i$: $d^{alpha}_i$ (alpha-blending depth) and $d^{mode}_i$ (mode-selection depth) [S1].
*   **Relative Difference Map $\Delta_i$:** Per-pixel relative difference between $d^{mode}_i$ and $d^{alpha}_i$ [S1].
*   **Floater Mask $F_i$:** Binary mask indicating pixels with relative difference exceeding the threshold [S1].

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
1. Initialize the average dip statistic $\bar{D} = 0.0$ [S1].
2. For each training view $i \in I$: render $d^{alpha}_i$ and $d^{mode}_i$, compute $\Delta_i$, compute the dip statistic for the distribution of $\Delta_i$, and add it to $\bar{D}$ [S1].
3. Compute the average dip statistic: $\bar{D} = \bar{D} / |I|$, where $|I|$ is the number of training views [S1].
4. For each training view $i \in I$: calculate the threshold $\tau_i$ using the percentile function with an exponential curve based on $\bar{D}$, generate the floater mask $F_i$ where $F_i = 1[\Delta_i > \tau_i]$, identify the indices of all Gaussians contributing to pixels where $F_i = 1$ (including the mode Gaussian), and remove these Gaussians from $G$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def advance_floater_pruning(G, training_cameras, a=0.97, b=-7.5):
    """
    Prunes floaters from 3D Gaussian Splatting representation G.
    """
    total_dip_statistic = 0.0
    num_views = len(training_cameras)

    # Pass 1: Compute average dip statistic across all views
    for i in range(num_views):
        cam = training_cameras[i]
        # Render alpha-blending depth: sum(T_i * alpha_i * d_i) per pixel [S1]
        d_alpha = render_alpha_blending_depth(G, cam)
        # Render mode-selection depth: depth of Gaussian with max alpha weight [S1]
        d_mode = render_mode_selection_depth(G, cam)

        # Compute relative difference map
        valid_mask = d_alpha > 1e-6
        delta_i = [0.0] * len(d_alpha)
        for idx in range(len(d_alpha)):
            if valid_mask[idx]:
                delta_i[idx] = (d_mode[idx] - d_alpha[idx]) / d_alpha[idx]

        # Compute dip statistic for the distribution of delta_i values [G]
        dip_stat = compute_dip_statistic([v for v in delta_i if v > 0])
        total_dip_statistic += dip_stat

    average_dip_statistic = total_dip_statistic / num_views

    # Pass 2: Apply pruning for each view
    gaussians_to_remove = set()
    for i in range(num_views):
        cam = training_cameras[i]
        d_alpha = render_alpha_blending_depth(G, cam)
        d_mode = render_mode_selection_depth(G, cam)

        valid_mask = d_alpha > 1e-6
        delta_i = [0.0] * len(d_alpha)
        for idx in range(len(d_alpha)):
            if valid_mask[idx]:
                delta_i[idx] = (d_mode[idx] - d_alpha[idx]) / d_alpha[idx]

        # Calculate threshold using exponential curve mapping dip statistic to percentile
        percentile_value = a * math.exp(b * average_dip_statistic)
        percentile_value = max(0.0, min(100.0, percentile_value))

        threshold = percentile(delta_i, percentile_value)

        # Generate floater mask
        floater_mask = [v > threshold for v in delta_i]

        # Identify Gaussians to remove by mapping masked pixels back to contributing Gaussians
        # This requires iterating over pixels in floater_mask and collecting all Gaussians
        # that contributed to those pixels during rasterization [G].
        for y, x in enumerate_pixels_where(floater_mask):
            gaussians_to_remove.update(get_contributing_gaussians(G, cam, x, y))

        # Remove Gaussians
        G.remove_gaussians(list(gaussians_to_remove))

    return G

def render_alpha_blending_depth(G, cam):
    # Not given in the sources.
    pass

def render_mode_selection_depth(G, cam):
    # Not given in the sources.
    pass

def compute_dip_statistic(values):
    # Not given in the sources.
    pass

def get_contributing_gaussians(G, cam, x, y):
    # Not given in the sources.
    pass
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $d^{alpha}_i$ is zero (e.g., background pixels), the relative difference $\Delta_i$ is undefined [G].
*   **Transmittance Threshold:** Floaters arise because 3DGS halts rasterization once a transmittance threshold is reached [S1]. Near-camera floaters deplete this budget early, preventing convergence at the correct depth [S1].
*   **Bimodal Distribution:** The method relies on the observation that floaters cause a bimodal distribution in $\Delta_i$ [S1]. If the scene does not exhibit this characteristic (e.g., no floaters), the dip statistic will be low, resulting in a high percentile threshold and minimal pruning [S1].
*   **Timing:** Sources conflict on execution timing; one states it is applied once after training completes, while another states it is performed at pre-set intervals during training [S1].

## Subtopics

### [Weighted Floater Removal](20006-weighted-floater-removal.md)
Weighted Floater Removal (WFR) is an explicit post-processing method to eliminate 3D Gaussian floaters in whole-scene reconstruction by verifying multi-view depth consistency. Unlike TSDF Fusion, which incurs high computational costs by building voxel volumes and extracting meshes, WFR directly adjusts Gaussian opacities based on accumulated weights, avoiding rendering holes while maintaining geometric accuracy. It specifically addresses floaters introduced after the current view's processing in the incremental PTF module, which the parent [Advance Floater Pruning](17398-advance-floater-pruning.md) fails to remove.

### Floater Reduction via Depth Standard Deviation
Floater Reduction via Depth Standard Deviation is a technique to remove blurry floaters from albedo images by analyzing the depth distribution of Gaussians contributing to each pixel [S2]. For each input view, it visits each pixel and accumulates all Gaussians until transmittance is saturated (reaching 'expected termination' depth) [S2]. It computes the standard deviation of the depths of the accumulated primitives for that pixel, increments a counter for primitives that have a depth less than $n$ standard deviations of the average depth, and removes primitives with a high counter value [S2]. This operation is performed before every densification step, which happens every 100 iterations from iteration 500 to iteration 15000 of the optimization [S2]. The parameter $n$ represents the number of standard deviations threshold for identifying floaters, with typical values not explicitly stated but described as satisfying the condition for several pixels for larger floaters [S2].

## Operating characteristics
*   **Time Complexity:** The method requires rendering depth maps for all training views twice (once for dip statistic calculation, once for pruning) and computing dip statistics [S1]. The complexity is linear with respect to the number of training views and pixels [G].
*   **Memory Complexity:** Requires storing depth maps and relative difference maps for all training views [G].
*   **Performance:** Effectively removes "background collapse" and floating artifacts in sparse-input settings [S1].
*   **Strengths:** Explicitly targets geometric inconsistencies (floaters) that are not addressed by opacity-based pruning [S1]. Adaptive thresholding based on scene-wide dip statistic allows it to handle varying levels of floater presence [S1].
*   **Limitations:** Only applicable after training [S1]. Relies on the assumption that floaters cause bimodal depth difference distributions [S1].

## Background
*   **Origin:** Introduced in the paper "SparseGS: Sparse View Synthesis Using 3D Gaussian Splatting" [S1].
*   **Context:** Developed to address artifacts in 3D Gaussian Splatting when trained on sparse input views [S1]. The method complements other regularization techniques like Unseen Viewpoint Regularization and Score Distillation Sampling [S1].
*   **History:** The concept of "floaters" in 3DGS refers to low-opacity Gaussians near the camera that cause misalignment between mode-selection and alpha-blending depths [S1]. This pruning heuristic was empirically derived by examining $\Delta_i$ distributions across various scenes [S1].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
[S2] Intrinsic decomposition and editing of 3D Gaussian splats, https://doi.org/10.1145/3804495, fetched 2026-10-27
