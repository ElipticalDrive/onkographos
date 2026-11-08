# Gaussian Insertion and Pruning

Also called: Monocular Gaussian Initialization

## Overview
Gaussian Insertion and Pruning is a densification strategy designed for monocular and RGB-D SLAM systems using 3D Gaussian Splatting, specifically addressing the initialization of new Gaussians from depth estimates and the removal of geometrically unstable primitives [S1, unofficial]. Unlike standard densification which splits existing Gaussians based on gradient norms, this method inserts new Gaussians at keyframes to capture newly visible scene elements and prunes those that fail multiview consistency checks [S1, unofficial]. It is distinct from sibling methods like [Score-based Densification](4705-score-based-densification.md) by relying on visibility windows and depth-based initialization rather than per-step gradient magnitude thresholds.

## How it works
**Inputs:**
*   Current keyframe window $W_k$ of size $|W_k|$ (e.g., 10 for Replica, 8 for TUM) [S1, unofficial].
*   Rendered depth map $D$ for the current frame [S1, unofficial].
*   Median depth $\hat{D}$ of the rendered image [S1, unofficial].
*   Depth standard deviation $\sigma_D$ [S1, unofficial].
*   Opacity threshold $\alpha_{thresh} = 0.7$ [S1, unofficial].
*   Visibility counter for Gaussians inserted in the last 3 keyframes [S1, unofficial].

**Outputs:**
*   Updated set of 3D Gaussians with new insertions and pruned entries.

**Data Structures:**
*   **Keyframe Window $W_k$:** A circular buffer storing the most recent $|W_k|$ keyframes.
*   **Gaussian Attributes:** Each Gaussian $\mathcal{G}_i$ stores mean $\mu$, covariance $\Sigma$, opacity $\alpha$, and insertion timestamp/keyframe index.
*   **Visibility Count:** An integer counter for each Gaussian, incremented when the Gaussian is used in rasterization and the ray's accumulated $\alpha$ has not reached 0.5 [S1, unofficial].

**Steps:**

1.  **Initialization of New Gaussians:**
    At every keyframe, new Gaussians are inserted to capture newly visible elements [S1, unofficial].
    *   **RGB-D Case:** If depth measurements are available, initialize Gaussian means $\mu_W$ by back-projecting the depth values directly [S1, unofficial].
    *   **Monocular Case:** Render depth $D$ at the current frame. For each pixel $p$:
        *   If $D_p$ exists (observed region): Sample depth from a normal distribution $N(D_p, 0.2\sigma_D)$ [S1, unofficial].
        *   If $D_p$ does not exist (unobserved region): Sample depth from $N(\hat{D}, 0.5\sigma_D)$, where $\hat{D}$ is the median of $D$ [S1, unofficial].
        *   Initialize the 3D mean $\mu_W$ using the sampled depth and camera intrinsics [S1, unofficial].

2.  **Visibility Tracking:**
    During rasterization, mark a Gaussian as visible from a view if it is used in the rasterization process and the ray's accumulated opacity $\alpha$ has not yet reached 0.5 [S1, unofficial]. Increment the visibility counter for these Gaussians.

3.  **Pruning Conditions:**
    Pruning is performed under two conditions:
    *   **Opacity Pruning:** Remove all Gaussians with opacity $\alpha < 0.7$ [S1, unofficial].
    *   **Visibility-Based Pruning:**
        *   Trigger only when the keyframe window $W_k$ is full [S1, unofficial].
        *   Identify Gaussians inserted within the last 3 keyframes [S1, unofficial].
        *   For each such Gaussian, check if it was observed by at least 3 other frames within the current window $W_k$ [S1, unofficial].
        *   If a Gaussian is unobserved by at least 3 other frames (i.e., visibility count is low relative to the window), prune it as geometrically unstable [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_insertion_and_pruning(keyframe_window, current_depth_map, median_depth, sigma_d, opacity_threshold=0.7):
    """
    Inserts new Gaussians based on depth and prunes unstable ones.
    
    Args:
        keyframe_window: List of recent keyframes (size |W_k|)
        current_depth_map: 2D array of rendered depth D
        median_depth: Scalar float, median of D
        sigma_d: Scalar float, depth standard deviation
        opacity_threshold: Float, default 0.7
        
    Returns:
        Updated list of Gaussians
    """
    new_gaussians = []
    
    # Step 1: Insert new Gaussians
    h, w = current_depth_map.shape
    for y in range(h):
        for x in range(w):
            d_p = current_depth_map[y, x]
            
            # Check if depth is valid (not NaN or zero)
            if is_valid_depth(d_p):
                # Observed region: sample from N(D_p, 0.2 * sigma_d)
                sampled_depth = sample_normal(d_p, 0.2 * sigma_d)
            else:
                # Unobserved region: sample from N(median_depth, 0.5 * sigma_d)
                sampled_depth = sample_normal(median_depth, 0.5 * sigma_d)
            
            # Back-project to 3D mean mu_W
            mu_W = back_project(x, y, sampled_depth, camera_intrinsics)
            
            # Create new Gaussian with low initial opacity (standard 3DGS practice)
            # Note: Specific initial opacity value not given in [S1, unofficial], using standard logit(-1.386) [G]
            new_g = Gaussian(mean=mu_W, opacity=0.05, insertion_keyframe=current_keyframe_index)
            new_gaussians.append(new_g)
            
    # Add new Gaussians to the global set
    all_gaussians.extend(new_gaussians)
    
    # Step 2: Update visibility counts
    # This is typically done during rasterization in the main loop
    # For this function, we assume visibility counts are updated externally
    # or passed in. Here we simulate the check for pruning.
    
    # Step 3: Pruning
    
    # 3a: Opacity Pruning
    all_gaussians = [g for g in all_gaussians if g.opacity >= opacity_threshold]
    
    # 3b: Visibility-Based Pruning
    # Only perform if window is full
    if len(keyframe_window) == max_window_size:
        # Identify Gaussians inserted in the last 3 keyframes
        recent_gaussians = [g for g in all_gaussians 
                            if current_keyframe_index - g.insertion_keyframe <= 3]
        
        pruned_indices = []
        for g in recent_gaussians:
            # Count how many other frames in the window observed this Gaussian
            # Visibility is defined as: used in rasterization AND accumulated alpha < 0.5
            visibility_count = g.visibility_count_in_window
            
            # Prune if unobserved by at least 3 other frames
            # This implies visibility_count < (len(W_k) - 1 - 3) ? 
            # The source says: "unobserved by at least 3 other frames"
            # If window size is 10, and it's in the last 3, there are 7 other frames.
            # If it is unobserved by 3 of them, it was observed by at most 4.
            # The condition is: if (number of unobserved frames) >= 3
            # Let N_obs be observed count. Unobserved = (Total Other Frames) - N_obs
            # Condition: (Total Other Frames) - N_obs >= 3
            # => N_obs <= (Total Other Frames) - 3
            
            total_other_frames = len(keyframe_window) - 1
            if g.visibility_count_in_window <= total_other_frames - 3:
                pruned_indices.append(g.id)
                
        # Remove pruned Gaussians
        all_gaussians = [g for g in all_gaussians if g.id not in pruned_indices]
        
    return all_gaussians

def is_valid_depth(d):
    return d is not None and d > 0

def sample_normal(mean, std):
    # Standard normal sampling
    import random
    return random.gauss(mean, std)

def back_project(x, y, depth, intrinsics):
    # Simple back-projection assuming pinhole camera
    # fx, fy, cx, cy from intrinsics
    X = (x - intrinsics.cx) * depth / intrinsics.fx
    Y = (y - intrinsics.cy) * depth / intrinsics.fy
    Z = depth
    return np.array([X, Y, Z])
```

**Parameters:**
*   $\sigma_D$: Depth standard deviation. Not given in the sources. [G] Typically derived from depth sensor noise or estimated from the depth map statistics.
*   $|W_k|$: Keyframe window size. 10 for Replica, 8 for TUM [S1, unofficial].
*   Opacity Threshold: 0.7 [S1, unofficial].
*   Variance Multipliers: 0.2 for observed regions, 0.5 for unobserved regions [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Incorrect Depth Estimates:** In monocular settings, rendered depth may be incorrect. The high variance initialization ($0.5\sigma_D$) for unobserved regions accounts for this, but may lead to initial geometric instability [S1, unofficial].
*   **Window Full Condition:** Visibility-based pruning only occurs when $W_k$ is full. If the window is not full, unstable Gaussians may persist longer [S1, unofficial].
*   **Visibility Definition:** Visibility is strictly defined as being used in rasterization with accumulated $\alpha < 0.5$. This handles occlusions without additional heuristics [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Insertion is $O(N_{pixels})$ per keyframe for monocular depth-based initialization. Pruning is $O(N_{Gaussians})$ per step.
*   **Memory Complexity:** Requires storing visibility counts for each Gaussian and maintaining the keyframe window $W_k$. Buffer capacity must be sufficient to handle insertions before compaction.
*   **Strengths:** Effective for SLAM systems where depth is estimated or available. Handles monocular depth uncertainty via variance-based initialization. Reduces floaters through visibility-based pruning.
*   **Limitations:** Relies on the quality of rendered depth. Inaccurate depth leads to initial noise that must be pruned. Not suitable for static scene reconstruction without a keyframe window mechanism.
*   **When to Use:** Monocular or RGB-D SLAM with 3D Gaussian Splatting.
*   **When Not to Use:** Static scene reconstruction with known camera poses (use standard densification).

## Background
Gaussian Insertion and Pruning was introduced in "Gaussian Splatting SLAM" [S1, unofficial]. It addresses the challenge of initializing and maintaining 3D Gaussians in a SLAM context, where camera poses are estimated and depth is not ground-truth. The method combines depth-based initialization with visibility-based pruning to ensure geometric consistency across the keyframe window.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
