# Gaussian Insertion and Pruning

Also called: Monocular Gaussian Initialization

## Overview
Gaussian Insertion and Pruning is a dynamic densification and culling strategy used in monocular and RGB-D Gaussian Splatting SLAM systems to manage the 3D Gaussian population during online mapping [S1, unofficial]. It addresses the problem of initializing new geometry from single-view depth estimates and removing geometrically unstable "floaters" that violate multiview consistency [S1, unofficial]. Unlike standard offline 3DGS training which uses gradient-based splitting, this method relies on visibility tracking across a sliding keyframe window to determine insertion variance and pruning eligibility [S1, unofficial].

## How it works
**Inputs:**
*   Current rendered depth map $D$ for the current camera pose.
*   Keyframe window $W_k$ containing recent keyframes.
*   Current set of 3D Gaussians $\mathcal{G}$.
*   Hyperparameters: $\sigma_D$ (depth noise standard deviation), $\alpha_{min} = 0.7$ (opacity threshold), $N_{obs} = 3$ (minimum observations), $N_{recent} = 3$ (recent keyframe window size for pruning check) [S1, unofficial].

**Outputs:**
*   Updated set of 3D Gaussians $\mathcal{G}'$ with new insertions and pruned primitives.

**Data Structures:**
*   **Keyframe Window $W_k$:** A fixed-size buffer of recent keyframes. Size $|W_k|$ is typically 10 for Replica and 8 for TUM datasets [S1, unofficial].
*   **Visibility Buffer:** A boolean flag or counter per Gaussian indicating if it was used in rasterization and contributed to the accumulated alpha ($\alpha < 0.5$) in a specific view [S1, unofficial].

**Steps:**

1.  **Initialization of New Gaussians:**
    When a new keyframe is added, new Gaussians are inserted to capture newly visible elements. The mean position $\mu_W$ is initialized based on the rendered depth $D$:
    *   **Case A (Depth Available):** For a pixel $p$ where rendered depth $D_p$ exists, sample the depth from a normal distribution:
        $$ z \sim \mathcal{N}(D_p, 0.2\sigma_D) $$
        The 3D mean $\mu_W$ is back-projected using this sampled depth [S1, unofficial].
    *   **Case B (Unobserved Regions):** For pixels without depth estimates, initialize around the median depth $\hat{D}$ of the rendered image with higher variance:
        $$ z \sim \mathcal{N}(\hat{D}, 0.5\sigma_D) $$
        The 3D mean $\mu_W$ is back-projected using this sampled depth [S1, unofficial].

2.  **Visibility Tracking:**
    During rasterization, a Gaussian is marked as "visible" from a view if it is used in the rasterization process and the ray's accumulated alpha has not yet reached 0.5. This handles occlusions without additional heuristics [S1, unofficial].

3.  **Visibility-Based Pruning:**
    Pruning is performed only when the keyframe window $W_k$ is full.
    *   Identify Gaussians inserted within the last 3 keyframes.
    *   Check if these Gaussians were observed (marked visible) by at least 3 other frames within the current window $W_k$.
    *   If a Gaussian is unobserved by at least 3 other frames, it is considered geometrically unstable and is pruned [S1, unofficial].

4.  **Opacity-Based Pruning:**
    *   Prune all Gaussians with opacity $\alpha < 0.7$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_insertion_and_pruning(current_depth_map, keyframe_window, gaussians, sigma_D):
    """
    current_depth_map: Rendered depth map D for current frame.
    keyframe_window: List of recent keyframes W_k.
    gaussians: List of active 3D Gaussian objects.
    sigma_D: Depth noise standard deviation.
    """
    
    # Step 1: Insert new Gaussians based on depth
    new_gaussians = []
    median_depth = median(current_depth_map)
    
    for pixel in current_depth_map:
        if pixel.has_depth:
            # Case A: Depth available
            sampled_z = sample_normal(pixel.depth, 0.2 * sigma_D)
        else:
            # Case B: Unobserved region
            sampled_z = sample_normal(median_depth, 0.5 * sigma_D)
        
        # Back-project to 3D mean
        mu_W = back_project_3d(pixel.u, pixel.v, sampled_z)
        
        # Create new Gaussian with initial parameters
        g_new = Gaussian(mean=mu_W, covariance=initial_covariance, opacity=0.1, ...)
        g_new.insertion_keyframe_id = current_keyframe_id
        new_gaussians.append(g_new)
        
    gaussians.extend(new_gaussians)
    
    # Step 2: Visibility Tracking (Implicit in rasterization)
    # Gaussians are marked visible if used in rasterization and accumulated_alpha < 0.5
    
    # Step 3: Visibility-Based Pruning (Only if window is full)
    if keyframe_window.is_full():
        pruned_indices = []
        for i, g in enumerate(gaussians):
            # Check if inserted within last 3 keyframes
            if is_recent_insertion(g, current_keyframe_id, window_size=3):
                # Count observations in current window
                obs_count = count_observations(g, keyframe_window)
                
                # Prune if unobserved by at least 3 other frames
                # Note: "unobserved by at least 3" implies obs_count < (Total_Window_Size - 3)
                # Or strictly following text: "unobserved by at least 3 other frames"
                # Assuming window size W, if it is NOT seen in 3 frames, it is pruned.
                if obs_count < (len(keyframe_window) - 3):
                    pruned_indices.append(i)
        
        # Remove pruned Gaussians
        for i in reversed(pruned_indices):
            gaussians.pop(i)

    # Step 4: Opacity-Based Pruning
    pruned_indices = []
    for i, g in enumerate(gaussians):
        if g.opacity < 0.7:
            pruned_indices.append(i)
            
    for i in reversed(pruned_indices):
        gaussians.pop(i)
        
    return gaussians
```

## Operating characteristics
*   **Time Complexity:** Insertion is $O(N_{pixels})$ per keyframe. Pruning is $O(N_{Gaussians} \cdot |W_k|)$ as it requires checking visibility across the window.
*   **Memory Complexity:** Requires storing visibility counts for each Gaussian across the keyframe window.
*   **Strengths:** Effectively removes geometrically unstable floaters in monocular settings where depth is noisy or missing. Simple to implement.
*   **Limitations:** Relies on accurate rendered depth estimates. The opacity threshold (0.7) is high compared to standard 3DGS, potentially removing valid but low-opacity geometry. The visibility check is coarse (binary per frame).
*   **When to use:** Monocular or RGB-D SLAM systems using Gaussian Splatting for mapping.
*   **When not to use:** Offline 3DGS reconstruction where gradient-based densification is preferred, or when high-precision depth sensors are available and floaters are less of an issue.

## Background
This method originates from Gaussian Splatting SLAM, specifically the work by Liu et al. (2023) [S1, unofficial]. It was designed to handle the challenges of online mapping with Gaussians, where traditional offline densification strategies are not directly applicable due to the streaming nature of the data and the lack of strong priors in monocular settings. The method integrates depth initialization and visibility-based culling to maintain a coherent 3D structure.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
