# CT-NeRF

Also called: CT-NeRF Training Pipeline, Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory

## Overview
CT-NeRF is an incremental reconstruction optimization pipeline that jointly optimizes camera poses and a neural radiance field (NeRF) from sequential RGB images without prior pose or depth input [S1, unofficial]. It is specifically designed to handle complex trajectories involving large rotations where existing methods fail [S1, unofficial]. Unlike its siblings, CT-NeRF escapes local minima by constructing a pose graph with edges between neighboring camera poses (enforcing inter-pose consistency) and by using reprojection loss derived from dense pixel-level correspondences, rather than relying solely on photometric loss [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A sequence of $n$ RGB images $I = \{I_1, I_2, ..., I_n\}$ and a pre-trained dense correspondence network (DKM) [S1, unofficial].
*   **Outputs:** Optimized camera poses $P = \{P_1, ..., P_n\}$ and a NeRF model $\Theta$ representing the 3D scene [S1, unofficial].

**Data Structures**
*   **NeRF Model $\Theta$:** An MLP mapping 3D position $x \in \mathbb{R}^3$ and view direction $d \in \mathbb{R}^3$ to color $c \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}$ [S1, unofficial].
*   **Camera Pose $P_i$:** A $3 \times 4$ matrix $[R, t]$ where $R \in SO(3)$ and $t \in \mathbb{R}^3$, representing the camera-to-world transformation [S1, unofficial].
*   **Intrinsic Matrix $K$:** Standard camera intrinsic parameters [S1, unofficial].
*   **Correspondence Set $M$:** A set of tuples $m = (q, p, \alpha)$ where $q \in I_1$ is a pixel in image 1, $p \in I_2$ is the corresponding pixel in image 2, and $\alpha \in (0, 1]$ is the confidence score [S1, unofficial].
*   **Sparse Correspondence Set $M_{sparse}$:** A subset of $M$ containing $N_m$ samples with confidence $\alpha > t_m$ [S1, unofficial].

**Formulas**
*   **Camera Projection:** $p = \pi(x, P, K)$, where $x \in \mathbb{R}^3$ is a space point and $p \in \mathbb{R}^2$ is a pixel [S1, unofficial].
*   **Volume Rendering (Color):**
    $$ \tilde{I}_i(q) = \int_{s_{near}}^{s_{far}} T(s) \sigma(r(s)) c(r(s), d) ds $$
    where $r(s) = o + ds$, $o, d$ are obtained by backprojecting pixel $q$ using $P_i$ and $K$, and transmittance $T(s) = \exp\left(-\int_{s_{near}}^{s} \sigma(r(h)) dh\right)$ [S1, unofficial].
*   **Volume Rendering (Depth):**
    $$ \tilde{D}_i(q) = \int_{s_{near}}^{s_{far}} T(s) \sigma(r(s)) s ds $$
    [S1, unofficial].
*   **Photometric Loss:**
    $$ L_{phot} = \frac{1}{n} \sum_{i=1}^{n} \|\tilde{I}_i(P_i, \Theta) - I_i\|_2^2 $$
    [S1, unofficial].
*   **Reprojection Loss:**
    For a correspondence $(q, p, \alpha)$ between $I_1$ and $I_2$:
    1.  Render depth $z(q; \Theta)$ for pixel $q$ in $I_1$ using Eq. (5).
    2.  Backproject $q$ to 3D point $x$ using $P_1, K, z(q; \Theta)$.
    3.  Reproject $x$ to $I_2$ using $P_2, K$ to get $p'$.
    4.  Calculate geometric distance:
        $$ L_{rp}(I_1, I_2) = \frac{1}{N_s} \sum_{s=1}^{N_s} \alpha_s \| \pi(\pi^{-1}(q, P_1, K, z(q; \Theta)), P_2, K) - p \| $$
        [S1, unofficial].

**Parameters**
*   **Learning Rates:** Network $lr_\Theta$ decays from $1 \times 10^{-3}$ to $1 \times 10^{-4}$; Pose $lr_{\hat{P}}$ decays from $3 \times 10^{-3}$ to $1 \times 10^{-5}$ [S1, unofficial].
*   **Sampling:** Inverse sampling of 128 points along each ray with range $[1, 0)$ [S1, unofficial].
*   **Batch Size:** 1024 [S1, unofficial].
*   **Correspondence Sampling:** $N_m = 10,000$ correspondences with $\alpha > t_m = 0.2$ are selected as $M_{sparse}$ [S1, unofficial].
*   **Iteration Sampling:** $N_s = \lfloor \frac{256}{|E|} \rfloor$ correspondences are randomly fetched from $M_{sparse}$ per iteration, where $|E|$ is the number of images in the current optimization set [S1, unofficial].
*   **Positional Encoding:** Coarse-to-fine strategy controlled by parameter $\alpha_{pe}$, linearly adjusted from iteration 20K to 100K during post-optimization [S1, unofficial].
*   **Iteration Counts (NeRFBuster):** $N_{init}=3$, $N_{window}=4$, $\beta_{init}=2000$, $\beta_{tracking}=100$, $\beta_{window}=200$, $\beta_{global}=500$, $\beta_{post}=200,000$ [S1, unofficial].

**Algorithm Steps**
The pipeline consists of five sequential parts [S1, unofficial]:

1.  **Scene Initialization:** Initialize the NeRF $\Theta$ and poses $P$ using a small set of $N_{init}$ images. Optimize for $\beta_{init}$ iterations.
2.  **Tracking:** For a new image $I_{new}$, estimate a rough initial pose $P_{new}$ using the previous pose $P_{prev}$ and local feature matching (implied by "tracking" and DKM usage) [S1, unofficial].
3.  **Window Optimization:** Optimize poses and scene for a local window of images (including $I_{new}$ and $N_{window}$ previous frames) to refine local structure and pose consistency. Optimize for $\beta_{window}$ iterations.
4.  **Global Optimization (Bundle Adjustment):** Incorporate all currently added images $E_{global} = \{I_1, ..., I_i\}$ into the optimization set. Simultaneously optimize all poses and the scene using both photometric loss and reprojection loss between neighboring frames. Optimize for $\beta_{global}$ iterations.
5.  **Post Optimization:** After all frames are added, iteratively refine the entire scene and all poses. Gradually reduce learning rates and increase positional encoding frequency (BARF strategy) until convergence ($\beta_{post}$ iterations) [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ct_nerf_pipeline(images, K, dkm_network):
    """
    images: List of RGB images [I_1, ..., I_n]
    K: Camera intrinsic matrix
    dkm_network: Pre-trained dense correspondence network
    """
    # 1. Initialization
    theta = initialize_nerf()
    poses = [None] * len(images)
    
    # Initialize first N_init frames
    init_indices = range(0, N_init)
    poses[0] = identity_pose() # Or estimated from init set
    for i in range(1, N_init):
        poses[i] = estimate_rough_pose(poses[i-1], images[i-1], images[i])
        
    optimize_scene_and_poses(theta, poses, init_indices, K, dkm_network, iterations=beta_init)
    
    # 2. Incremental Processing
    for i in range(N_init, len(images)):
        # Tracking: Rough pose for new frame
        poses[i] = estimate_rough_pose(poses[i-1], images[i-1], images[i])
        
        # Window Optimization: Local window of size N_window + 1
        window_start = max(0, i - N_window)
        window_indices = range(window_start, i + 1)
        optimize_scene_and_poses(theta, poses, window_indices, K, dkm_network, iterations=beta_window)
        
        # Global Optimization: All frames so far
        global_indices = range(0, i + 1)
        optimize_scene_and_poses(theta, poses, global_indices, K, dkm_network, iterations=beta_global)
        
    # 3. Post Optimization
    all_indices = range(0, len(images))
    optimize_scene_and_poses_post(theta, poses, all_indices, K, dkm_network, 
                                  lr_theta_decay=True, lr_pose_decay=True, 
                                  pe_schedule='barf', iterations=beta_post)

def optimize_scene_and_poses(theta, poses, indices, K, dkm_network, iterations):
    """
    Standard optimization step using photometric and reprojection losses.
    """
    for step in range(iterations):
        # Sample batch of rays for NeRF
        rays = sample_rays(indices, batch_size=1024)
        
        # Render color and depth
        colors, depths = render_volume(theta, rays, poses)
        
        # Photometric Loss
        loss_phot = mean_squared_error(colors, ground_truth_colors(rays))
        
        # Reprojection Loss
        loss_rp = 0
        # Fetch sparse correspondences for the current window/global set
        # DKM generates dense correspondences M between adjacent images
        # We sample N_s from M_sparse
        sparse_corrs = sample_correspondences(indices, dkm_network, N_s)
        
        for q, p, alpha in sparse_corrs:
            # Get depth of q in its source image
            depth_q = render_depth(theta, q, poses[source_image_index(q)])
            # Backproject to 3D
            x_3d = backproject(q, poses[source_image_index(q)], K, depth_q)
            # Reproject to target image
            p_prime = project(x_3d, poses[target_image_index(q, p)], K)
            # Geometric distance
            loss_rp += alpha * distance(p_prime, p)
            
        loss_rp /= N_s
        
        # Total Loss
        loss = loss_phot + lambda_rp * loss_rp
        
        # Backpropagate and update theta and poses[indices]
        update_parameters(theta, poses, indices, loss)

def estimate_rough_pose(prev_pose, img_prev, img_curr):
    """
    Estimates initial pose for new frame.
    Uses DKM to find correspondences and solve PnP or similar.
    """
    # DKM provides dense correspondences M between img_prev and img_curr
    M = dkm_network(img_prev, img_curr)
    # Filter high confidence
    M_sparse = [m for m in M if m.confidence > 0.2]
    # Solve for pose using RANSAC or least squares on M_sparse
    # Initialize with prev_pose as prior if needed
    return solve_pose(M_sparse, K, prev_pose)
```

## Operating characteristics
*   **Time Complexity:** The method is iterative and incremental. Each step involves NeRF rendering (expensive) and optimization. Global optimization scales with the number of images $n$ and the number of correspondences. Post-optimization runs for a large number of iterations (e.g., 200K) [S1, unofficial].
*   **Memory Complexity:** Stores the NeRF MLP, all camera poses, and dense correspondence maps between adjacent frames. The correspondence set $M$ can be large (full-resolution pixel-level), requiring significant memory or on-the-fly generation [S1, unofficial].
*   **Performance:** Evaluated on NeRFBuster and Free-Dataset. Outperforms CF-NeRF, BARF, and Nope-NeRF in pose estimation accuracy and novel view synthesis for complex trajectories [S1, unofficial].
*   **Strengths:** Handles large rotations and complex trajectories; avoids local minima via inter-pose constraints and reprojection loss; produces smooth view transitions unlike BARF which may overfit [S1, unofficial].
*   **Limitations:** Requires dense correspondence network (DKM); sensitive to initialization; computationally intensive due to global optimization and post-optimization phases [S1, unofficial].
*   **When to use:** Scenes with complex camera trajectories, large rotations, and no prior pose information.
*   **When not to use:** Simple forward-facing scenes (BARF or standard NeRF may suffice); real-time applications due to high computational cost [S1, unofficial].

## Background
CT-NeRF was proposed to address the limitations of existing NeRF-based pose estimation methods (like CF-NeRF and BARF) which struggle with complex trajectories and local minima [S1, unofficial]. It builds on the BARF framework for coarse-to-fine positional encoding and the incremental strategy of CF-NeRF, but introduces a pose graph with inter-pose edges and reprojection loss to enforce geometric consistency [S1, unofficial]. The method is implemented based on the BARF framework [S1, unofficial].

## Sources
[S1] CT-NeRF: Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory, https://doi.org/10.48550/arxiv.2404.13896, fetched 2026-10-26 (unofficial)
