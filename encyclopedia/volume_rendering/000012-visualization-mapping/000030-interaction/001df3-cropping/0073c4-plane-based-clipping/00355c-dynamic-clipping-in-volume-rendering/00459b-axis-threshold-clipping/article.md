# Axis Threshold Clipping

Also called: Gaussian Axis Clipping

## Overview
Axis Threshold Clipping is a dynamic clipping method for 3D Gaussian Splatting that discards Gaussians whose projected screen-space bounding quad exceeds a maximum axis length threshold $\tau_\lambda$ [S1]. It solves the performance bottleneck of processing large, distant Gaussians that fall outside the view frustum but have significant screen-space extent, specifically targeting mobile hardware constraints [S1].

## How it works
**Inputs:**
- A list of 3D Gaussians, each defined by a mean position $\mu$, a covariance matrix $\Sigma$, and an opacity $\alpha$ [G].
- A camera projection matrix $P$ and view matrix $V$ combined into a single transformation matrix [G].
- A clipping threshold $\tau_\lambda$ (scalar) [S1].

**Outputs:**
- A subset of Gaussians eligible for rasterization [G].

**Data Structures:**
- Gaussian attributes: $\mu \in \mathbb{R}^3$, $\Sigma \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite), $\alpha \in [0, 1]$ [G].
- Clipping threshold: $\tau_\lambda \in \mathbb{R}^+$ [S1].

**Steps:**
1. For each Gaussian $i$ with mean $\mu_i$ and covariance $\Sigma_i$:
   a. Project the mean $\mu_i$ to normalized device coordinates (NDC) using the camera matrices [G].
   b. Compute the 2D covariance matrix $\Sigma'_i$ in screen space by transforming $\Sigma_i$ with the projection Jacobian $J$ at $\mu_i$: $\Sigma'_i = J \Sigma_i J^T$ [G].
   c. Compute the axis-aligned bounding box (AABB) extents of the projected Gaussian in screen pixels [G]. The extents along the screen x and y axes are $\sigma_{x,i} = k \sqrt{\Sigma'_{i,xx}}$ and $\sigma_{y,i} = k \sqrt{\Sigma'_{i,yy}}$, where $k$ is a confidence factor (typically 3) [G].
   d. Determine the maximum axis length: $L_{max, i} = \max(\sigma_{x,i}, \sigma_{y,i})$ [G].
   e. If $L_{max, i} > \tau_\lambda$, discard Gaussian $i$ [S1].
   f. Otherwise, keep Gaussian $i$ for rendering [S1].

**Parameters:**
- $\tau_\lambda$: The clipping threshold for the maximum axis of the screen-space quad. Typical value: 40 pixels [S1].

**Edge Cases and Pitfalls:**
- Gaussians with a mean outside the view frustum but with large splats inside the view frustum are skipped if their projected quad exceeds $\tau_\lambda$ [S1].
- Precision issues may arise when computing the Jacobian for Gaussians very close to the camera or at extreme depths [G].
- The threshold $\tau_\lambda$ must be tuned to the target resolution and hardware capabilities [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def axis_threshold_clipping(gaussians, camera_matrix, tau_lambda):
    """
    gaussians: List of dicts with 'mu' (3,), 'Sigma' (3,3), 'alpha' (float)
    camera_matrix: 4x4 list of lists representing combined view-projection matrix (row-major)
    tau_lambda: float, clipping threshold in screen units
    Returns: List of kept Gaussians
    """
    kept = []
    k = 3.0  # confidence factor for 3-sigma extent [G]
    
    for g in gaussians:
        mu = g['mu']
        Sigma = g['Sigma']
        
        # Project mean to NDC
        w_cam = sum(camera_matrix[3][c] * mu[c] for c in range(3)) + camera_matrix[3][3]
        if w_cam <= 0.0:
            continue
        ndc_x = (sum(camera_matrix[0][c] * mu[c] for c in range(3)) + camera_matrix[0][3]) / w_cam
        ndc_y = (sum(camera_matrix[1][c] * mu[c] for c in range(3)) + camera_matrix[1][3]) / w_cam
        
        # Compute Jacobian of projection at mu
        J = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
        for r in range(2):
            num = sum(camera_matrix[r][c] * mu[c] for c in range(3)) + camera_matrix[r][3]
            for c in range(3):
                d_num_dc = camera_matrix[r][c]
                d_den_dc = camera_matrix[3][c]
                J[r][c] = (d_num_dc * w_cam - num * d_den_dc) / (w_cam ** 2)
                
        # Transform 3D covariance to 2D screen space: Sigma_2d = J @ Sigma @ J^T
        Sigma_2d = [[0.0, 0.0], [0.0, 0.0]]
        for i in range(2):
            for j in range(2):
                val = 0.0
                for l in range(3):
                    for m in range(3):
                        val += J[i][l] * Sigma[l][m] * J[j][m]
                Sigma_2d[i][j] = val
                    
        # Compute extents in screen pixels
        extent_x = k * math.sqrt(max(0.0, Sigma_2d[0][0]))
        extent_y = k * math.sqrt(max(0.0, Sigma_2d[1][1]))
        
        max_axis = max(extent_x, extent_y)
        
        if max_axis <= tau_lambda:
            kept.append(g)
            
    return kept
```

## Subtopics

### Automatic Axis Aligned Cutting Tool
A clipping tool for volume rendering that divides the volume into eight octants and hides those located between the viewpoint and the Region of Interest (ROI) to allow fast and intuitive data observation [S2]. It also supports a sphere-based cutting method with 26 sectors (6 for half, 12 for quarter, 8 for eighth cuts) to discard octants occluding the origin based on the viewpoint's sector [S2]. It uses a mask volume with 1/10th resolution and 10x voxel size, where voxels with mask value 0 are discarded in a GLSL shader [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of Gaussians, as each Gaussian is checked once against the threshold [G].
- **Memory Complexity:** $O(1)$ additional memory per Gaussian for the check [G].
- **Performance:** On mobile VR (Quest3), using $\tau_\lambda = 40$ results in 19 FPS compared to 6 FPS without the threshold for the Bike scene with 790k splats [S1].
- **Strengths:** Significantly improves rendering performance on mobile hardware by skipping expensive processing of large, distant Gaussians [S1].
- **Limitations:** May discard Gaussians that are technically inside the view frustum but have large extents, potentially causing visual artifacts if $\tau_\lambda$ is too low [G].

## Background
Axis Threshold Clipping was introduced in the context of real-time Gaussian Splatting in VR to address performance issues on mobile hardware [S1]. The Automatic Axis Aligned Cutting Tool, a related concept for traditional volume rendering, was introduced by Christian Rieder et al. in 2008 for medical visualization [S2].

## Sources
[S1] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
[S2] High-Quality Multimodal Volume Visualization of Intracerebral Pathological Tissue, https://doi.org/10.2312/vcbm/vcbm08/167-176, fetched 2026-10-26
