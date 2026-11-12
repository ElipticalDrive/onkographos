# Gaussian Insertion and Pruning

Also called: Monocular Gaussian Initialization

## Overview
Gaussian Insertion and Pruning is a densification strategy used in monocular and RGB-D SLAM systems to maintain a coherent 3D Gaussian representation by dynamically adding primitives to capture newly visible scene elements and removing geometrically unstable ones [S1, unofficial]. It distinguishes itself from standard 3D Gaussian Splatting training by integrating visibility checks across a sliding window of keyframes to prune Gaussians that fail to maintain multiview consistency, rather than relying solely on opacity thresholds or gradient magnitudes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current rendered depth map $D$, current Gaussian set $\mathcal{G}$, keyframe window $W_k$ (size $|W_k|$), and a counter for recent insertions.
*   **Outputs:** Updated Gaussian set $\mathcal{G}'$ with new Gaussians inserted and unstable ones pruned.

**Data Structures:**
*   **Keyframe Window ($W_k$):** A fixed-size buffer storing recent keyframes. Typical sizes are $|W_k| = 10$ for Replica and $|W_k| = 8$ for TUM datasets [S1, unofficial].
*   **Visibility Buffer:** A Gaussian is marked as visible from a view if it was used in rasterization and the ray's accumulated opacity $\alpha$ has not yet reached 0.5 [S1, unofficial].

**Parameters:**
*   $\sigma_D$: Standard deviation of depth noise/uncertainty.
*   $\alpha_{\text{min}} = 0.7$: Opacity threshold for pruning [S1, unofficial].
*   $N_{\text{insert}} = 3$: Number of recent keyframes to check for insertion stability [S1, unofficial].
*   $N_{\text{obs}} = 3$: Minimum number of other frames required to observe a Gaussian for it to remain [S1, unofficial].

**Steps:**

1.  **Depth Estimation:**
    *   If RGB-D data is available, back-project depth measurements to initialize Gaussian means $\mu_W$ [S1, unofficial].
    *   If monocular, render the depth map $D$ for the current frame [S1, unofficial].

2.  **Gaussian Insertion (Initialization):**
    For each pixel $p$ in the current frame:
    *   If depth $D_p$ exists: Sample the depth $z$ from a normal distribution $N(D_p, 0.2\sigma_D)$ [S1, unofficial].
    *   If depth $D_p$ is missing (unobserved region): Calculate the median depth $\hat{D}$ of the rendered image $D$. Sample the depth $z$ from $N(\hat{D}, 0.5\sigma_D)$ [S1, unofficial].
    *   Initialize the new Gaussian mean $\mu_W$ at the back-projected 3D position corresponding to $z$ and pixel coordinates. Initialize covariance and opacity based on standard 3DGS priors [S1, unofficial].

3.  **Visibility-Based Pruning:**
    *   Only perform this step once the keyframe window $W_k$ is full [S1, unofficial].
    *   Identify Gaussians inserted within the last $N_{\text{insert}}$ keyframes.
    *   For each such Gaussian, count how many *other* frames in $W_k$ (excluding the insertion frames) observed it (using the visibility definition: rasterized and accumulated $\alpha < 0.5$) [S1, unofficial].
    *   If the count is less than $N_{\text{obs}}$, mark the Gaussian for removal [S1, unofficial].

4.  **Opacity-Based Pruning:**
    *   Remove all Gaussians in $\mathcal{G}$ where opacity $\alpha < \alpha_{\text{min}}$ (0.7) [S1, unofficial].

5.  **Buffer Management:**
    *   Compact the Gaussian buffer to remove pruned entries, ensuring contiguous memory access for subsequent rasterization steps [Parent].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_insertion_and_pruning(current_frame, keyframe_window, gaussians, sigma_D):
    """
    current_frame: Object containing rendered depth map D and pixel coordinates.
    keyframe_window: List of recent keyframes, size |Wk|.
    gaussians: List of Gaussian objects with fields: mean, cov, opacity, visibility_flags (dict mapping frame_id -> bool).
    sigma_D: Depth noise standard deviation.
    """
    # Step 1 & 2: Insertion
    new_gaussians = []
    median_depth = median(current_frame.D)
    
    for pixel in current_frame.pixels:
        depth_val = current_frame.D[pixel]
        
        if depth_val is not None:
            # Sample from observed depth with low variance
            z = sample_normal(depth_val, 0.2 * sigma_D)
        else:
            # Sample from median depth with high variance for unobserved regions
            z = sample_normal(median_depth, 0.5 * sigma_D)
            
        # Back-project to 3D mean
        mu_W = back_project_3d(pixel, z)
        
        # Initialize Gaussian with standard 3DGS defaults for cov/opacity
        g = Gaussian(mean=mu_W, cov=init_cov(), opacity=init_opacity())
        new_gaussians.append(g)
        
    # Add new Gaussians to the main set
    gaussians.extend(new_gaussians)
    
    # Step 3: Visibility-Based Pruning (only if window is full)
    if len(keyframe_window) >= get_window_size():
        # Identify Gaussians inserted in the last 3 keyframes
        recent_insertion_ids = get_recent_insertion_ids(gaussians, n=3)
        
        for g_id in recent_insertion_ids:
            g = gaussians[g_id]
            obs_count = 0
            
            # Check visibility in other frames in the window
            for kf in keyframe_window:
                if kf.id not in recent_insertion_ids: # Exclude insertion frames
                    # Check if this Gaussian was visible in this frame
                    # Visibility: rasterized and accumulated alpha < 0.5
                    if is_visible(g, kf):
                        obs_count += 1
            
            # Prune if not observed by at least 3 other frames
            if obs_count < 3:
                mark_for_removal(g_id)
                
    # Step 4: Opacity-Based Pruning
    for g in gaussians:
        if g.opacity < 0.7:
            mark_for_removal(g.id)
            
    # Step 5: Compaction
    gaussians = compact_buffer(gaussians)
    
    return gaussians
```

## Operating characteristics
*   **Time Complexity:** Insertion is $O(P)$ where $P$ is the number of pixels. Pruning is $O(G \cdot |W_k|)$ where $G$ is the number of recently inserted Gaussians, as each must be checked against other frames in the window.
*   **Memory Complexity:** Increases with insertion; decreases with pruning. The window size $|W_k|$ directly limits the memory required for visibility tracking.
*   **Strengths:** Effectively removes "floaters" and geometrically unstable Gaussians in monocular settings where depth initialization is noisy [S1, unofficial].
*   **Limitations:** Requires a full keyframe window before pruning begins, delaying cleanup in early stages. Relies on accurate rendered depth; incorrect depth estimates lead to initial noise that must be pruned [S1, unofficial].

## Background
This method originates from **Gaussian Splatting SLAM**, a monocular and RGB-D SLAM system that adapts 3D Gaussian Splatting for real-time mapping [S1, unofficial]. It was introduced to address the challenge of initializing Gaussians without strong depth priors in monocular setups and maintaining consistency across a sliding window of keyframes.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
