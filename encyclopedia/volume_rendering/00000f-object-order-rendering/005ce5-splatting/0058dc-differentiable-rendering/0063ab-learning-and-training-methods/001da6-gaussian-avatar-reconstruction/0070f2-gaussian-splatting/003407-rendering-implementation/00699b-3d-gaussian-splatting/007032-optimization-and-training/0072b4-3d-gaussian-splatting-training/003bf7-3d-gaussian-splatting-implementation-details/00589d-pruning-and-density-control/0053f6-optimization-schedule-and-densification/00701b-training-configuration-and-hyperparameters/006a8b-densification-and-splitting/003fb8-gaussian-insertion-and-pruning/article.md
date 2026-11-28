# Gaussian Insertion and Pruning

Also called: Monocular Gaussian Initialization

## Overview
Gaussian Insertion and Pruning is a density control strategy for 3D Gaussian Splatting SLAM that manages the population of Gaussians during optimization to handle newly visible scene elements and remove geometrically unstable outliers [S1, unofficial]. It differs from static densification methods by integrating insertion and pruning directly into the keyframe-based optimization loop, using rendered depth and visibility checks to maintain a coherent 3D structure without requiring strong external priors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Current keyframe window $W_k$, rendered depth map $D$, current Gaussian set $\mathcal{G}$, opacity values $\alpha$, and iteration counters.
- **Outputs:** Updated Gaussian set $\mathcal{G}'$ with new Gaussians inserted and unstable ones pruned.

**Data Structures:**
- **Keyframe Window $W_k$:** A fixed-size buffer of recent keyframes. Size $|W_k|$ is typically 10 for Replica and 8 for TUM datasets [S1, unofficial].
- **Gaussian Attributes:** Each Gaussian $g_i$ has mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and color $c_i$.
- **Visibility Counter:** A count of how many frames in $W_k$ have observed a specific Gaussian. A Gaussian is considered "observed" by a view if it was used in rasterization and the ray's accumulated $\alpha$ has not yet reached 0.5 [S1, unofficial].

**Parameters:**
- **Depth Variance Scaling:** $\sigma_D$ (standard deviation of depth noise/uncertainty).
- **Insertion Variance Factors:** 0.2 for pixels with depth, 0.5 for unobserved regions [S1, unofficial].
- **Pruning Thresholds:**
  - Opacity threshold: 0.7 [S1, unofficial].
  - Visibility threshold: Unobserved by at least 3 other frames [S1, unofficial].
  - Age threshold: Gaussians inserted within the last 3 keyframes [S1, unofficial].

**Steps:**
1. **Initialization of New Gaussians:**
   - If depth measurements are available, initialize Gaussian means $\mu_W$ by back-projecting the depth [S1, unofficial].
   - In the monocular case, render the depth map $D$ for the current frame.
   - For each pixel $p$ where depth $D_p$ exists:
     - Sample depth from a normal distribution: $N(D_p, 0.2\sigma_D)$ [S1, unofficial].
     - Initialize $\mu_W$ at this sampled depth.
   - For pixels without depth estimates (unobserved regions):
     - Calculate $\hat{D}$, the median of the rendered depth map $D$ [S1, unofficial].
     - Sample depth from a normal distribution: $N(\hat{D}, 0.5\sigma_D)$ [S1, unofficial].
     - Initialize $\mu_W$ at this sampled depth.
   - Assign initial covariance and opacity based on standard Gaussian Splatting initialization practices [G].

2. **Visibility-Based Pruning (Conditional):**
   - Only perform this step once the keyframe window $W_k$ is full [S1, unofficial].
   - For each Gaussian $g_i$ inserted within the last 3 keyframes:
     - Count the number of frames in $W_k$ where $g_i$ was observed (as defined above) [S1, unofficial].
     - If $g_i$ is unobserved by at least 3 other frames, mark $g_i$ for pruning [S1, unofficial].

3. **Opacity-Based Pruning:**
   - For all Gaussians in $\mathcal{G}$:
     - If opacity $\alpha_i < 0.7$, mark $g_i$ for pruning [S1, unofficial].

4. **Execution:**
   - Remove all marked Gaussians from $\mathcal{G}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_insertion_and_pruning(keyframe_window, current_frame, gaussians, sigma_D):
    """
    keyframe_window: List of recent keyframes (size |W_k|)
    current_frame: The new keyframe being processed
    gaussians: List of current 3D Gaussians
    sigma_D: Standard deviation of depth uncertainty
    """
    new_gaussians = []
    
    # Render depth map for current frame
    depth_map = render_depth(current_frame, gaussians)
    
    # Calculate median depth for unobserved regions
    valid_depths = [d for d in depth_map.flatten() if d is not None]
    median_depth = median(valid_depths) if valid_depths else 0.0
    
    # Insert new Gaussians based on depth
    for pixel in current_frame.image_pixels:
        depth_val = depth_map[pixel]
        if depth_val is not None:
            # Sample from N(D_p, 0.2 * sigma_D)
            sampled_depth = sample_normal(depth_val, 0.2 * sigma_D)
            mean_3d = back_project(pixel, sampled_depth)
        else:
            # Sample from N(median_D, 0.5 * sigma_D)
            sampled_depth = sample_normal(median_depth, 0.5 * sigma_D)
            mean_3d = back_project(pixel, sampled_depth)
        
        # Initialize Gaussian with mean_3d, default covariance, and initial opacity
        new_gaussian = create_gaussian(mean=mean_3d, opacity=0.1) # [G] Initial opacity low
        new_gaussian.insertion_keyframe = current_frame.id
        new_gaussians.append(new_gaussian)
    
    # Add new Gaussians to the scene
    gaussians.extend(new_gaussians)
    
    # Pruning
    to_prune = set()
    
    # Only prune if window is full
    if len(keyframe_window) >= keyframe_window.max_size:
        for g in gaussians:
            # Check opacity
            if g.opacity < 0.7:
                to_prune.add(g)
                continue
            
            # Check visibility-based pruning for recent Gaussians
            if (current_frame.id - g.insertion_keyframe) <= 3:
                # Count observations in window
                observation_count = 0
                for kf in keyframe_window:
                    if is_observed(g, kf):
                        observation_count += 1
                
                # If unobserved by at least 3 other frames
                # Note: "unobserved by at least 3 other frames" implies 
                # observed_count <= (|W_k| - 1 - 3) or similar logic depending on 
                # exact definition of "other". The source says "unobserved by at least 3 other frames".
                # Assuming W_k includes current frame and recent ones.
                # If a Gaussian is in the last 3 keyframes, we check if it's seen in the rest.
                # Let's interpret "unobserved by at least 3 other frames" as:
                # The number of frames in W_k that did NOT observe it is >= 3.
                unobserved_count = len(keyframe_window) - observation_count
                if unobserved_count >= 3:
                    to_prune.add(g)
    
    # Remove pruned Gaussians
    gaussians = [g for g in gaussians if g not in to_prune]
    
    return gaussians

def is_observed(gaussian, keyframe):
    """
    A Gaussian is visible if used in rasterization and accumulated alpha < 0.5
    """
    # [G] This requires access to the rasterization buffer or a re-render
    # to check if the Gaussian contributed to the final pixel color before alpha thresholding.
    # Simplified check: if the Gaussian projects within the image and has non-zero opacity.
    # For precise implementation, re-render or store visibility flags.
    return gaussian.visible_in(keyframe)
```

**Edge Cases and Pitfalls:**
- **Incorrect Depth Estimates:** In monocular settings, rendered depth may be inaccurate. The variance initialization ($0.2\sigma_D$ and $0.5\sigma_D$) accounts for this, but incorrect depth can lead to Gaussians being placed in empty space, requiring pruning [S1, unofficial].
- **Window Size:** Pruning only occurs when $W_k$ is full. If the window is not full, unstable Gaussians may persist, potentially causing artifacts [S1, unofficial].
- **Visibility Definition:** The definition of "observed" (accumulated $\alpha < 0.5$) handles occlusions but requires careful implementation of the rasterization pass to track this metric [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Insertion is $O(N_p)$ where $N_p$ is the number of pixels in the current frame. Pruning is $O(N_g \cdot |W_k|)$ where $N_g$ is the number of Gaussians, as each recent Gaussian must be checked against all frames in the window.
- **Memory Complexity:** Increases with the number of inserted Gaussians. Pruning helps bound the total number of Gaussians.
- **Strengths:** Effectively removes geometrically unstable Gaussians in SLAM scenarios without requiring strong priors. Handles monocular depth uncertainty through variance-based initialization.
- **Limitations:** Relies on the accuracy of rendered depth for initialization. Pruning is delayed until the keyframe window is full, which may allow temporary accumulation of outliers.
- **When to Use:** Ideal for 3D Gaussian Splatting SLAM systems, particularly in monocular or RGB-D settings where depth estimates are noisy or incomplete.

## Background
Gaussian Insertion and Pruning was introduced in "Gaussian Splatting SLAM" (2023) as a method to maintain a coherent 3D structure during simultaneous localization and mapping using 3D Gaussian Splatting [S1, unofficial]. It addresses the challenge of depth error and surface bumps/protrusions caused by original densification and splitting strategies in Gaussian Splatting [S1, unofficial]. The method is particularly relevant for monocular SLAM, where depth information is estimated rather than measured.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
