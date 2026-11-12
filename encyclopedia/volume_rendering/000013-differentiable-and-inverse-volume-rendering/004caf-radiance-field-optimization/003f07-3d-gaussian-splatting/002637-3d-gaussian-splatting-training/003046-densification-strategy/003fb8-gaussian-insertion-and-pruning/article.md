# Gaussian Insertion and Pruning

Also called: Monocular Gaussian Initialization

## Overview
Gaussian Insertion and Pruning is a dynamic densification and culling strategy used in monocular and RGB-D Gaussian Splatting SLAM systems to maintain a coherent 3D structure while optimizing camera pose and scene representation [S1, unofficial]. It addresses the problem of accumulating geometric errors and redundant primitives in online reconstruction by periodically inserting new Gaussians to capture newly visible regions and pruning those that are geometrically unstable or have low opacity [S1, unofficial]. Unlike static densification methods that rely solely on gradient magnitude, this method leverages a sliding window of keyframes and visibility consistency checks to ensure multi-view consistency, making it distinct from sibling methods like Score-based Densification which focus on per-primitive error metrics [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current set of Gaussians $\mathcal{G}$, current camera pose, rendered depth map $D$ (monocular) or sensor depth (RGB-D), keyframe window $W_k$, and visibility buffers.
*   **Outputs:** Updated set of Gaussians $\mathcal{G}'$ with new insertions and pruned elements.

**Data Structures:**
*   **Keyframe Window ($W_k$):** A fixed-size buffer storing recent keyframes. Size $|W_k|$ is typically 10 for Replica and 8 for TUM datasets [S1, unofficial].
*   **Visibility Buffer:** A boolean flag per Gaussian indicating if it was used in rasterization and if the ray's accumulated $\alpha$ has not yet reached 0.5, handling occlusions implicitly [S1, unofficial].

**Parameters:**
*   $\sigma_D$: Standard deviation of depth noise/uncertainty.
*   $\alpha_{\min}$: Opacity threshold for pruning, set to 0.7 [S1, unofficial].
*   $N_{\text{insert}}$: Number of keyframes to look back for insertion age (3) [S1, unofficial].
*   $N_{\text{obs}}$: Minimum number of other frames required to observe a Gaussian for survival (3) [S1, unofficial].

**Steps:**

1.  **Initialization of New Gaussians:**
    At each keyframe, new Gaussians are inserted to capture newly visible elements. The mean position $\mu_W$ is initialized based on depth:
    *   **RGB-D Case:** Back-project the measured depth to initialize $\mu_W$ [S1, unofficial].
    *   **Monocular Case:** Render the depth map $D$ at the current frame.
        *   For pixels $p$ where rendered depth $D_p$ exists, sample the depth from a normal distribution:
            $$ \mu_W \sim \mathcal{N}(D_p, 0.2\sigma_D) $$
        *   For unobserved regions (no depth estimate), sample from the median depth $\hat{D}$ of the rendered image with higher variance:
            $$ \mu_W \sim \mathcal{N}(\hat{D}, 0.5\sigma_D) $$
        [S1, unofficial]

2.  **Visibility-Based Pruning (Geometric Stability):**
    *   Check Gaussians inserted within the last 3 keyframes.
    *   If a Gaussian is unobserved by at least 3 other frames in the current window $W_k$, it is considered geometrically unstable and is pruned [S1, unofficial].
    *   This pruning step is only performed once the keyframe window $W_k$ is full [S1, unofficial].

3.  **Opacity-Based Pruning:**
    *   Prune all Gaussians with opacity $\alpha < 0.7$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_insertion_and_pruning(Gaussians, KeyframeWindow, RenderedDepth, sigma_D):
    """
    Performs insertion and pruning for Gaussian Splatting SLAM.
    
    Args:
        Gaussians: List of Gaussian primitives.
        KeyframeWindow: List of recent keyframes.
        RenderedDepth: 2D array of rendered depth values for current frame.
        sigma_D: Depth noise standard deviation.
        
    Returns:
        Updated list of Gaussians.
    """
    
    # 1. Insertion Phase
    # Determine median depth for unobserved regions
    valid_depths = [d for d in RenderedDepth.flatten() if d > 0]
    if valid_depths:
        median_depth = median(valid_depths)
    else:
        median_depth = 0.0 # Fallback if no depth available
        
    new_gaussians = []
    for pixel in RenderedDepth:
        if pixel.depth > 0:
            # Sample from N(D_p, 0.2 * sigma_D)
            depth_sample = random_normal(pixel.depth, 0.2 * sigma_D)
            mean_3d = back_project(pixel.u, pixel.v, depth_sample)
        else:
            # Sample from N(median_D, 0.5 * sigma_D)
            depth_sample = random_normal(median_depth, 0.5 * sigma_D)
            mean_3d = back_project(pixel.u, pixel.v, depth_sample)
            
        # Initialize new Gaussian with mean_3d, default covariance, and low opacity
        new_gaussians.append(create_gaussian(mean_3d, opacity=0.1))
        
    Gaussians.extend(new_gaussians)
    
    # 2. Pruning Phase
    
    # A. Visibility-based pruning for recent insertions
    if len(KeyframeWindow) >= KeyframeWindow.max_size:
        recent_insertion_threshold = 3 # Keyframes
        min_observation_count = 3
        
        for g in Gaussians:
            if g.is_inserted_recently(recent_insertion_threshold):
                # Count observations in the window
                obs_count = 0
                for kf in KeyframeWindow:
                    if g.is_visible_in(kf):
                        obs_count += 1
                
                if obs_count < min_observation_count:
                    g.mark_for_deletion()
                    
    # B. Opacity-based pruning
    for g in Gaussians:
        if g.opacity < 0.7:
            g.mark_for_deletion()
            
    # Remove marked Gaussians
    Gaussians = [g for g in Gaussians if not g.marked_for_deletion()]
    
    return Gaussians
```

**Edge Cases and Pitfalls:**
*   **Incorrect Depth Estimates:** In monocular settings, rendered depth may be incorrect. The variance terms ($0.2\sigma_D$ and $0.5\sigma_D$) account for this, but excessive noise may lead to initial geometric instability, requiring the visibility pruning to remove these "floaters" [S1, unofficial].
*   **Window Size:** Pruning only occurs when the window is full. If the window is not full, unstable Gaussians may persist temporarily, potentially affecting tracking accuracy [S1, unofficial].
*   **Occlusion Handling:** Visibility is defined by rasterization usage and accumulated $\alpha < 0.5$, which naturally handles occlusions without explicit heuristics [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Insertion is $O(P)$ where $P$ is the number of pixels in the rendered depth map. Pruning is $O(N \cdot |W_k|)$ where $N$ is the number of Gaussians and $|W_k|$ is the window size, as each recent Gaussian must be checked against all frames in the window.
*   **Memory Complexity:** Increases linearly with the number of inserted Gaussians and decreases with pruning. The keyframe window adds constant overhead relative to the scene size.
*   **Strengths:** Effectively removes geometrically unstable Gaussians in SLAM contexts, preventing drift and floaters. Simple to implement using standard rasterization visibility buffers.
*   **Limitations:** Relies on the accuracy of rendered depth in monocular mode. The fixed thresholds (opacity 0.7, 3 keyframes/3 frames) may not be optimal for all scenes.
*   **When to use:** Monocular or RGB-D SLAM systems using 3D Gaussian Splatting where online reconstruction and camera tracking are required.
*   **When not to use:** Static scene reconstruction where global optimization is feasible, or when high-precision depth sensors are available and monocular depth estimation is unreliable.

## Background
Gaussian Insertion and Pruning was introduced in "Gaussian Splatting SLAM" (2023) as part of a monocular and RGB-D SLAM pipeline based on 3D Gaussian Splatting [S1, unofficial]. It adapts the densification principles of 3DGS for online settings, where the scene is not fully known a priori. The method emphasizes maintaining a coherent 3D structure through a sliding window of keyframes and visibility consistency checks, distinguishing it from offline densification strategies.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
