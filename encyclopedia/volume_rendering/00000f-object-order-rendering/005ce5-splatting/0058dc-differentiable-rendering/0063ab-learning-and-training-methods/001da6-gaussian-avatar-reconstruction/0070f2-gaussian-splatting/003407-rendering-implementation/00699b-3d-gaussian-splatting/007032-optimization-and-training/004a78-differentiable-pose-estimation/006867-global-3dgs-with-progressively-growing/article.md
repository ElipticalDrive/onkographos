# Global 3DGS with Progressively Growing

Also called: Progressive Global 3DGS, Sequential Global 3DGS

## Overview
Global 3DGS with Progressively Growing is a sequential reconstruction method that learns a global 3D Gaussian Splatting (3DGS) model by incrementally integrating frames, using estimated relative poses to update the scene representation and densify Gaussians to cover new views [S1]. It differs from standard global 3DGS by avoiding the accumulation of noisy relative pose errors across long sequences; instead, it grows the model frame-by-frame, aligning densification intervals with the addition of new frames to ensure continuous coverage of unobserved regions [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A sequence of images $\{I_t\}$, where $t$ is the frame index.
- **Outputs:** A global set of 3D Gaussians $\mathcal{G}$ with attributes (position, covariance, opacity, spherical harmonics coefficients) and estimated camera poses for all frames.

**Data Structures:**
- **3D Gaussian Set $\mathcal{G}$:** A collection of $K$ Gaussians. Each Gaussian $k$ has:
  - Position $\mu_k \in \mathbb{R}^3$
  - Rotation $R_k \in SO(3)$ (represented as quaternion or rotation matrix)
  - Scale $S_k \in \mathbb{R}^3$
  - Opacity $\alpha_k \in [0, 1]$
  - Spherical Harmonics coefficients $c_k$
- **Camera Poses:** For each frame $t$, a pose $P_t$ consisting of rotation $q_t \in SO(3)$ and translation $t_t \in \mathbb{R}^3$.

**Parameters:**
- $N$: Number of optimization iterations per step. This value is set equal to the interval between adding new frames and performing densification [S1].
- Initial learning rate for pose optimization: $10^{-5}$, decaying to $10^{-6}$ [S1].
- Densification threshold: Based on the average magnitude of viewspace position gradients [S1].

**Steps:**
1. **Initialization:** Start with the first frame $I_t$. Initialize a set of 3D Gaussian points (e.g., via depth estimation or feature matching, though the source specifies starting from frame $I_t$ with an orthogonal camera pose) [S1]. Set the initial camera pose for $I_t$ as orthogonal [S1].
2. **Relative Pose Estimation:** Use Local 3DGS to estimate the relative camera pose between the current frame $I_t$ and the next frame $I_{t+1}$ [S1].
3. **Global Update:** Update the global 3D Gaussian set $\mathcal{G}$ and all its attributes over $N$ iterations. The optimization uses the estimated relative pose and the two observed frames ($I_t, I_{t+1}$) as inputs [S1].
4. **Pose Representation:** Camera poses are optimized in the representation of quaternion rotation $q \in SO(3)$ and translation vector $t \in \mathbb{R}^3$ [S1].
5. **Progression:** When the next frame $I_{t+2}$ becomes available, repeat the process: estimate the relative pose between $I_{t+1}$ and $I_{t+2}$, and infer the relative pose between $I_t$ and $I_{t+2}$ [S1].
6. **Densification:** Densify the global 3DGS every $N$ steps (aligned with the pace of adding new frames) [S1]. Candidates for densification are determined by the average magnitude of viewspace position gradients. Gaussians with large average magnitude of viewspace position gradients indicate under-reconstruction and are split or cloned [S1].
7. **Opacity Reset:** Keep resetting the opacity of Gaussians until the end of the training process to enable integration of new frames into the existing model [S1].
8. **Continuous Growth:** Continue growing the 3D Gaussian points (densification) until the end of the input sequence, rather than stopping in the middle of training [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def global_3dgs_progressive_growth(image_sequence):
    # Initialize global Gaussian set with the first frame
    # Assume orthogonal pose for the first frame
    G = initialize_gaussians(image_sequence[0], pose='orthogonal')
    
    # Set optimization parameters
    N = 1000  # Number of steps for adding new frames and densification intervals
    lr_pose_init = 1e-5
    lr_pose_final = 1e-6
    
    for t in range(1, len(image_sequence)):
        I_t = image_sequence[t-1]
        I_next = image_sequence[t]
        
        # Estimate relative pose between I_t and I_next using Local 3DGS
        relative_pose = local_3dgs_estimate_pose(I_t, I_next)
        
        # Update global 3DGS over N iterations
        for step in range(N):
            # Decay learning rate for pose optimization
            lr_pose = lr_pose_init * (lr_pose_final / lr_pose_init) ** (step / N)
            
            # Optimize global Gaussians and poses using I_t, I_next and relative_pose
            # The relative pose helps align I_next to the global coordinate system
            optimize_global_3dgs(G, I_t, I_next, relative_pose, lr_pose)
            
            # Reset opacity to allow integration of new frames
            reset_opacity(G)
        
        # Densify Gaussians every N steps
        # Target Gaussians with large average magnitude of viewspace position gradients
        densify_gaussians(G, threshold=get_gradient_threshold(G))
        
    return G
```

**Edge Cases and Pitfalls:**
- **Noisy Relative Poses:** Accumulated noise in relative poses can dramatically impact the optimization of the global 3DGS for the whole scene [S1]. The progressive approach mitigates this by limiting the propagation of errors [S1].
- **Under-reconstruction:** Unobserved frames may contain regions not yet well reconstructed. The optimization tries to move Gaussians to correct this with large gradient steps, which is why densification targets these regions [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of frames, as each frame is processed sequentially with $N$ iterations of optimization and densification [S1].
- **Memory Complexity:** Linear with respect to the number of Gaussians, which grows progressively with the number of frames [S1].
- **Performance:** Evaluated on Tanks and Temples and CO3D-V2 datasets [S1]. Achieves photo-realistic reconstruction and accurate camera pose estimation [S1].
- **Strengths:** Avoids error accumulation from noisy relative poses; continuously covers new views; integrates new frames effectively [S1].
- **Limitations:** Requires sequential processing; densification must be carefully timed with frame addition [S1].

## Background
- **Origin:** Proposed in "COLMAP-Free 3D Gaussian Splatting" (CVPR 2024) [S1].
- **History:** Builds on 3D Gaussian Splatting [17] and Local 3DGS [S1]. Addresses issues with global 3DGS optimization when relative poses are noisy [S1].
- **Usage:** Used for novel view synthesis and pose estimation on real-world datasets like Tanks and Temples and CO3D-V2 [S1].

## Sources
[S1] COLMAP-Free 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.01965, fetched 2026-10-26
