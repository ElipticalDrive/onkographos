# GS-SLAM


## Overview
GS-SLAM is a Simultaneous Localization and Mapping (SLAM) method that reconstructs 3D scenes and estimates camera poses using 3D Gaussian Splatting [S1]. It distinguishes itself from sibling methods like SplaTAM by employing an adaptive 3D Gaussian expanding strategy that dynamically adds new Gaussians and deletes unreliable ones based on captured depth and rendered opacity, rather than using a fixed densification mask or view-independent colors [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB-D image frames (color and depth), initial camera poses (or monocular initialization), and an initial set of 3D Gaussians.
*   **Outputs:** Optimized camera poses, updated 3D Gaussian parameters (means, covariances, opacities, spherical harmonics), and a rendered RGB-D image.

**Data Structures:**
The scene is represented as a collection of $N$ 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$. Each Gaussian $g_i$ contains:
*   Mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$.
*   Covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$, parameterized by rotation $R_i \in SO(3)$ and scale $S_i \in \mathbb{R}^3$.
*   Opacity $\alpha_i \in [0, 1]$.
*   Spherical harmonics coefficients $c_{i,lm}$ for view-dependent color [S1][G].

**Algorithm Steps:**
1.  **Initialization:** Initialize the Gaussian set $\mathcal{G}$ and camera poses. [G] Typically initialized from sparse point clouds or previous frames.
2.  **Rendering:** Project current Gaussians to 2D using differentiable volume rendering (alpha blending) to produce rendered RGB and depth images [S1].
3.  **Loss Computation:** Compute the loss function $L$ combining photometric and geometric terms:
    $$ L = L_{rgb} + \lambda L_{depth} + L_{reg} $$
    where $L_{rgb}$ is typically $L_1$ or LPIPS loss, $L_{depth}$ is the depth error, and $L_{reg}$ includes regularization [S1].
4.  **Adaptive Expanding (Add):** Add new 3D Gaussians to the training stage. The specific criteria for "new" Gaussians are determined by the adaptive strategy, which identifies regions needing more detail [S2, unofficial].
5.  **Unreliable Deletion (Remove):** Delete unreliable Gaussians. A Gaussian is considered unreliable if its rendered opacity is low or if it does not align well with captured depth values [S1][S2, unofficial].
6.  **Optimization:** Update Gaussian parameters $\mathcal{G}$ and camera poses $T_{CW}$ by minimizing $L$ [S1].
7.  **Loop:** Repeat for subsequent frames.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GS-SLAM Core Loop
def gs_slam_loop(frames, initial_gaussians):
    gaussians = initial_gaussians
    for frame in frames:
        # 1. Render current state
        rendered_rgb, rendered_depth = render(gaussians, frame.camera_pose)
        
        # 2. Compute Loss
        # L_rgb: Difference between rendered_rgb and frame.rgb
        # L_depth: Difference between rendered_depth and frame.depth
        # L_reg: Regularization terms (e.g., isotropic scale)
        loss = compute_loss(rendered_rgb, rendered_depth, frame)
        
        # 3. Optimize
        # Update Gaussians and Camera Pose
        gaussians, frame.camera_pose = optimize(loss, gaussians, frame.camera_pose)
        
        # 4. Adaptive Expanding Strategy
        # Add new Gaussians where needed
        gaussians = add_new_gaussians(gaussians, frame)
        
        # 5. Delete Unreliable Gaussians
        # Remove based on captured depth and rendered opacity
        gaussians = delete_unreliable_gaussians(gaussians, frame)
        
    return gaussians, frame.camera_pose

def add_new_gaussians(gaussians, frame):
    # Identify regions with high error or low coverage
    # Add new Gaussians to these regions
    # Implementation details for the specific adaptive strategy are not fully detailed in the source
    # [G] Typically involves splitting or cloning existing Gaussians in high-error areas
    return gaussians

def delete_unreliable_gaussians(gaussians, frame):
    for g in gaussians:
        # Check rendered opacity and depth consistency
        if g.opacity < threshold or depth_error(g, frame) > depth_threshold:
            remove(g)
    return gaussians
```

**Parameters:**
*   $\lambda$: Weight for depth loss. Not given in the sources.
*   Opacity/Depth thresholds for deletion: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Duplicate Densification:** GS-SLAM's adaptive strategy aims to avoid the duplicate densification issues seen in other methods like SplaTAM [S1].
*   **Unreliable Gaussians:** Failure to correctly identify and remove unreliable Gaussians can lead to noise and artifacts in the reconstruction [S2, unofficial].

## Operating characteristics
**Performance:**
On the Replica dataset, GS-SLAM achieves a PSNR of 31.56, SSIM of 0.97, and LPIPS of 0.094 [S1]. This is competitive with other SLAM methods such as Co-SLAM (PSNR 30.24) but lower than SplaTAM (PSNR 34.11) and GaussianSplattingSLAM (PSNR 38.90) [S1].

**Complexity:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Adaptive expansion allows for dynamic scene reconstruction and efficient memory usage by removing unreliable Gaussians [S1][S2, unofficial].
*   **Limitations:** Performance is lower than some sibling methods like GaussianSplattingSLAM [S1]. The specific implementation details of the adaptive strategy are not fully elaborated in the provided sources.

## Background
GS-SLAM was proposed in the context of 3D Gaussian Splatting for SLAM, aiming to improve upon implicit representation-based methods by allowing easier surface determination [S1][S2, unofficial]. It is cited as reference [180] in the survey "Recent advances in 3D Gaussian splatting" [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
