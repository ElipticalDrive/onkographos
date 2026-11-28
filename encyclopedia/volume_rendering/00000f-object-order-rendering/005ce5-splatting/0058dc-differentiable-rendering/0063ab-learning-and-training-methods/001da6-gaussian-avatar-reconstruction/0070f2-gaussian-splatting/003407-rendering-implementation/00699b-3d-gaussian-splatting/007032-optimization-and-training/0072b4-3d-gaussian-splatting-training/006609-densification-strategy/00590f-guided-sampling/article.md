# Guided Sampling

Also called: Sparse point cloud densification with depth guidance

## Overview
Guided Sampling is a densification strategy for 3D Gaussian Splatting that addresses blurring artifacts in regions insufficiently covered by the initial sparse point cloud, such as far content outside windows or flames [S1]. It distinguishes itself from standard gradient-based splitting by explicitly sampling new Gaussian positions along the camera ray depth range rather than relying solely on positional gradients, thereby ensuring coverage where gradient signals are weak or ambiguous [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of 3D Gaussians $\mathcal{G}$, camera parameters for the current training view, and the maximum depth $d$ present in that view [S1].
*   **Output:** A set of new Gaussian primitives added to $\mathcal{G}$, positioned along the sampled depth range [S1].

**Data Structures:**
*   **Gaussian Primitive:** Defined by mean position $\mu \in \mathbb{R}^3$, covariance $\Sigma$, opacity $\alpha$, and spherical harmonic coefficients $\mathbf{c}$ [Parent].
*   **Depth Range:** Interval $[L, U]$ where $L = s \cdot d$ and $U = 7.5 \cdot d$ [S1].

**Parameters:**
*   $d$: Maximum depth value present in the training view [S1].
*   $s$: Scaling factor for the lower bound of the sampling range. Typically set to $0.7$ for most scenes [S1].
*   Noise: Small random perturbation added to each depth sample [S1].

**Steps:**
1.  **Determine Depth Bounds:** Compute $L = s \cdot d$ and $U = 7.5 \cdot d$ [S1].
2.  **Sample Depth:** Draw $z$ uniformly from $[L, U]$ and add small random noise $\epsilon$ to obtain $z' = z + \epsilon$ [S1].
3.  **Project to 3D:** Back-project the 2D pixel coordinates $(u, v)$ using the centered-undistorted camera model and sampled depth $z'$ to compute $\mu_{new}$ [S1].
4.  **Initialize Gaussian:** Create $g_{new}$ with $\mu_{new}$, scale and rotation inherited from the parent or default settings, and opacity $\alpha$ initialized according to standard densification rules (e.g., halved if splitting) [Parent].
5.  **Add to Set:** Insert $g_{new}$ into $\mathcal{G}$ [S1].
6.  **Pruning:** After densification and guided sampling, Gaussians with small spatial opacity are gradually pruned to retain representative primitives [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def guided_sampling(camera_intrinsics, max_depth_d, scaling_factor_s=0.7, num_samples=1):
    """
    Generates new Gaussian positions using guided depth sampling.
    camera_intrinsics: dict with keys 'fx', 'fy', 'cx', 'cy', 'width', 'height' [G].
    """
    new_means = []
    lower_bound = scaling_factor_s * max_depth_d
    upper_bound = 7.5 * max_depth_d
    
    for _ in range(num_samples):
        # Step 2: Sample depth uniformly and add noise [S1]
        z_sample = np.random.uniform(lower_bound, upper_bound)
        noise = np.random.normal(0, 0.01) # Small random noise [S1]
        z_final = z_sample + noise
        
        # Step 3: Project to 3D using centered-undistorted pinhole model [G]
        # Pixel coordinates (u, v) are sampled uniformly across the image [G]
        u = np.random.randint(0, camera_intrinsics['width'])
        v = np.random.randint(0, camera_intrinsics['height'])
        
        fx = camera_intrinsics['fx']
        fy = camera_intrinsics['fy']
        cx = camera_intrinsics['cx']
        cy = camera_intrinsics['cy']
        
        # Standard back-projection for centered-undistorted coordinates
        x = (u - cx) * z_final / fx
        y = (v - cy) * z_final / fy
        z = z_final
        mu_new = np.array([x, y, z])
        
        new_means.append(mu_new)
    return new_means
```

**Edge Cases and Pitfalls:**
*   **Blurring in Uncovered Areas:** Guided sampling cannot fully eliminate blurring in areas insufficiently covered by the sparse point cloud, such as far content outside windows or flames [S1]. This occurs because accurate depth is unavailable, forcing Gaussians to spawn across a depth range [S1].
*   **Pruning of Correct Gaussians:** Gaussians spawned near the correct location may be pruned during subsequent training if they do not immediately minimize the loss, leading to instability [S1].
*   **Mitigation:** Leveraging depth priors from learning-based depth estimation methods can constrain the spawning range and improve accuracy [S1].

## Subtopics

### [Geometry-Texture-Aware Densification](20660-geometry-texture-aware-densification.md)
Geometry-Texture-Aware Densification (GeoTexDensifier) is a densification strategy for 3D Gaussian Splatting that utilizes geometry priors (monocular depth and normal maps) to guide splitting in weakly textured regions and texture gradients to control densification in fully textured areas, thereby avoiding the over-splitting of floaters in textureless regions. It distinguishes itself from sibling methods like Mini-Splatting-D by replacing the static contribution-area threshold with a texture-weighted statistic and validating split positions against geometric consistency, ensuring splats conform to actual surface tangents rather than drifting into background noise.

## Operating characteristics
*   **Strengths:** Alleviates blurring in regions with sparse initial coverage by explicitly sampling depth [S1].
*   **Limitations:** Cannot fully eliminate artifacts in challenging scenarios like far content outside windows or flames where depth is ambiguous [S1]. May spawn Gaussians in incorrect locations that are later pruned, adding computational overhead [S1].
*   **Performance:** The source provides performance metrics for the full Spacetime Gaussian Feature Splatting pipeline but does not isolate the cost of guided sampling alone [S1].

## Background
Guided Sampling is introduced in the context of Spacetime Gaussian Feature Splatting for Real-Time Dynamic View Synthesis [S1]. It is designed to handle dynamic scenes and challenging geometry where static point cloud initialization is insufficient [S1].

## Sources
[S1] Spacetime Gaussian Feature Splatting for Real-Time Dynamic View Synthesis, https://huggingface.co/papers/2312.16812, fetched 2026-10-30
