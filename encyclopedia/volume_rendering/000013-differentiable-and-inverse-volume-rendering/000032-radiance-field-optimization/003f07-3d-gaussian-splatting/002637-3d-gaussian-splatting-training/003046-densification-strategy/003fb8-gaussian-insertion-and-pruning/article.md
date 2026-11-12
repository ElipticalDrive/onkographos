# Gaussian Insertion and Pruning

Also called: Monocular Gaussian Initialization

## Overview
Gaussian Insertion and Pruning is a dynamic primitive management strategy used in monocular and RGB-D SLAM pipelines based on 3D Gaussian Splatting. It addresses the problem of maintaining a coherent 3D structure and capturing newly visible scene elements during camera motion, which static densification strategies cannot handle. Unlike sibling methods that focus on optimizing reconstruction error for static scenes, this method relies on temporal visibility consistency across a sliding window of keyframes to insert new Gaussians and prune geometrically unstable ones [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current camera pose, rendered depth map $D$, current Gaussian set $\mathcal{G}$, keyframe window $W_k$ (size $|W_k|$), and opacity threshold $\alpha_{thresh}$.
*   **Outputs:** Updated Gaussian set $\mathcal{G}'$ with new primitives inserted and unstable ones removed.

**Data Structures:**
*   **Keyframe Window ($W_k$):** A fixed-size buffer storing recent keyframes. Typical sizes are $|W_k|=10$ for Replica and $|W_k|=8$ for TUM datasets [S1, unofficial].
*   **Visibility Mask:** A Gaussian is marked visible from a view if it is used in rasterization and the ray's accumulated opacity $\alpha$ has not reached 0.5 [S1, unofficial].

**Parameters:**
*   **Depth Variance Scaling:** $\sigma_D$ is the standard deviation of depth noise/uncertainty.
*   **Insertion Variance:** $0.2\sigma_D$ for observed pixels, $0.5\sigma_D$ for unobserved pixels [S1, unofficial].
*   **Pruning Thresholds:** Opacity $< 0.7$; Visibility count $< 3$ frames; Age $> 3$ keyframes [S1, unofficial].

**Steps:**

1.  **Initialization of New Means ($\mu_W$):**
    At every keyframe, new Gaussians are inserted to capture newly visible elements. The mean position $\mu_W$ is initialized based on depth availability:
    *   **RGB-D Case:** Back-project depth measurements to initialize $\mu_W$ [S1, unofficial].
    *   **Monocular Case:** Render depth $D$ at the current frame.
        *   For pixel $p$ where rendered depth $D_p$ exists: Sample depth from a normal distribution $N(D_p, 0.2\sigma_D)$ [S1, unofficial].
        *   For unobserved regions (no depth estimate): Sample from $N(\hat{D}, 0.5\sigma_D)$, where $\hat{D}$ is the median depth of the rendered image [S1, unofficial].

2.  **Visibility-Based Pruning (Geometric Stability):**
    This step ensures that newly inserted Gaussians are geometrically consistent with the rest of the scene.
    *   **Condition:** Only performed once the keyframe window $W_k$ is full [S1, unofficial].
    *   **Check:** Identify Gaussians inserted within the last 3 keyframes.
    *   **Action:** If such a Gaussian is unobserved (not visible) by at least 3 other frames in the current window $W_k$, it is pruned. This removes geometrically unstable primitives that violate multiview consistency [S1, unofficial].

3.  **Opacity-Based Pruning:**
    *   **Action:** Prune all Gaussians in the scene with opacity $\alpha < 0.7$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_insertion_and_pruning(keyframe_window, current_depth_map, gaussians, sigma_D):
    # Step 1: Insert new Gaussians
    new_gaussians = []
    median_depth = median(current_depth_map)
    
    for pixel in current_depth_map:
        if pixel.has_depth:
            # Sample from observed depth with low variance
            depth_sample = sample_normal(pixel.depth, 0.2 * sigma_D)
        else:
            # Sample from median depth with high variance for unobserved regions
            depth_sample = sample_normal(median_depth, 0.5 * sigma_D)
        
        # Back-project or initialize mean based on depth_sample
        mu_W = back_project(pixel, depth_sample)
        new_gaussians.append(create_gaussian(mean=mu_W))
    
    gaussians.extend(new_gaussians)
    
    # Step 2: Visibility-Based Pruning (only if window is full)
    if len(keyframe_window) == keyframe_window.max_size:
        recent_gaussians = get_gaussians_inserted_in_last_n_keyframes(gaussians, n=3)
        for g in recent_gaussians:
            visibility_count = count_frames_where_gaussian_is_visible(g, keyframe_window)
            if visibility_count < 3:
                remove_gaussian(gaussians, g)
                
    # Step 3: Opacity-Based Pruning
    for g in gaussians:
        if g.opacity < 0.7:
            remove_gaussian(gaussians, g)
            
    return gaussians
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians and keyframes in the window for pruning checks. Insertion is proportional to the number of pixels sampled.
*   **Memory Complexity:** Increases with the number of inserted Gaussians; decreases with pruning. The keyframe window size is fixed.
*   **Strengths:** Effectively handles dynamic scene changes and occlusions in SLAM by leveraging temporal consistency. Prevents the accumulation of "floaters" in monocular settings where depth ambiguity is high.
*   **Limitations:** Relies on accurate depth estimation (rendered or measured). In monocular settings, initial positions may be incorrect, requiring optimization to resolve or pruning to remove. The opacity threshold (0.7) is aggressive and may remove valid but low-opacity primitives [S1, unofficial].
*   **When to use:** Monocular or RGB-D SLAM systems using 3D Gaussian Splatting where camera motion introduces new views. Not suitable for static scene reconstruction where densification is driven by error metrics rather than temporal visibility.

## Background
This method originates from **Gaussian Splatting SLAM**, a preprint that adapts 3D Gaussian Splatting for simultaneous localization and mapping. It was introduced to handle the challenges of monocular depth estimation and dynamic scene capture, where traditional densification strategies fail to maintain geometric consistency over time [S1, unofficial].

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
