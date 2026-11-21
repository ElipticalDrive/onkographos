# Conditional 3DGS Optimization and Rendering

Also called: 7DGS to Conditional 3DGS, Optimization and Rendering Pipeline

## Overview
Conditional 3DGS Optimization and Rendering extends 3D Gaussian Splatting to handle dynamic scenes by incorporating temporal and directional dependencies into the Gaussian parameters [S1]. It solves the problem of representing time-varying geometry and appearance by conditioning the Gaussian covariance and opacity on time $t$ and view direction $d$ [S1]. It differs from static 3DGS by using a conditional covariance matrix to drive densification and splitting, and from other dynamic variants by integrating these cues directly into the standard 3DGS rasterization pipeline with minimal architectural changes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of images (video) with known camera poses and timestamps.
*   **Output:** A set of 7D Gaussians (3D position, 3D covariance, 3D color, opacity) conditioned on time and view direction, rendered as a video sequence.

**Data Structures:**
The core data structure is the Gaussian, defined by:
*   Mean position $\mu \in \mathbb{R}^3$.
*   Conditional Covariance $\Sigma_{cond} \in \mathbb{R}^{7 \times 7}$, which encodes spatial, temporal, and angular correlations.
*   Color coefficients $c \in \mathbb{R}^3$.
*   Conditional Opacity $\alpha_{cond}$, modulated by time and direction.

**Method Steps:**

1.  **Covariance Decomposition for Densification:**
    To guide splitting and pruning, extract scale and rotation from the conditional covariance $\Sigma_{cond}$ using Singular Value Decomposition (SVD) [S1]:
    $$ \Sigma_{cond} = U D U^T $$
    where $U$ is an orthogonal matrix and $D$ is a diagonal matrix of singular values [S1].
    *   Define rotation matrix $R = U$.
    *   Define scale vector $S = \text{diag}(D)$ [S1].
    *   Ensure a right-handed coordinate system by adjusting the last column of $R$:
        $$ R_{:,3} = R_{:,3} \cdot \text{sign}(\det(R)) $$
        [S1]

2.  **Conditional Splitting Criteria:**
    Leverage temporal cues for Gaussian splitting in addition to spatial gradients [S1]. Quantify spatial-temporal correlation using the magnitude of the off-diagonal block $\Sigma_{pt}$ of $\Sigma_{cond}$, which captures interaction between spatial and temporal components [S1].
    *   Split a Gaussian if:
        1.  $|\Sigma_{pt}| > 0.05$ (relative to screen extent).
        2.  Normalized temporal scale (derived from $\Sigma_t$) $> 0.25$ [S1].
    This ensures regions with significant motion dynamics are densely represented [S1].

3.  **Rendering Pipeline:**
    The pipeline remains compatible with standard 3DGS [S1].
    *   Convert the 7DGS representation to a 3DGS-compatible format via conditional slicing [S1].
    *   Project the resulting conditional 3D Gaussian mean and covariance onto the image plane using standard perspective projection to yield 2D Gaussians [S1].
    *   Splat onto the image canvas using a differentiable rasterization routine [S1].
    *   Compute final pixel colors by aggregating contributions in a depth-aware, opacity-blended manner [S1].

4.  **Optimization and Pruning:**
    *   Use the same loss functions and optimizers as standard 3DGS [S1].
    *   Increase the minimum opacity threshold for pruning to $\tau_{min} = 0.01$ to compensate for the modulation of conditional opacity $\alpha_{cond}$ by time and view direction [S1].

5.  **Modulation Parameters:**
    *   Directional and temporal modulation parameters ($\lambda_d$ and $\lambda_t$) are initialized based on dataset characteristics:
        *   0.5 for 7DGS-PBR.
        *   0.05 for D-NeRF.
        *   0.1 for Technicolor [S1].
    *   These parameters become trainable after 15,000 iterations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompose_conditional_covariance(Sigma_cond):
    """
    Extracts scale and rotation from the conditional covariance matrix.
    Args:
        Sigma_cond: 7x7 conditional covariance matrix.
    Returns:
        R: 3x3 rotation matrix.
        S: 3D scale vector.
    """
    # SVD decomposition
    U, D, Vt = np.linalg.svd(Sigma_cond)
    
    # Define rotation and scale
    R = U[:, :3]  # Extract 3x3 spatial rotation block
    S = np.diag(D)[:3]  # Extract top 3 singular values as scale
    
    # Ensure right-handed coordinate system
    det_R = np.linalg.det(R)
    if det_R < 0:
        R[:, 2] = -R[:, 2]
        
    return R, S

def check_splitting_criteria(Sigma_cond, screen_extent):
    """
    Determines if a Gaussian should be split based on temporal cues.
    Args:
        Sigma_cond: 7x7 conditional covariance matrix.
        screen_extent: Normalization factor for screen size.
    Returns:
        bool: True if splitting criteria are met.
    """
    # Extract off-diagonal block Sigma_pt (interaction between spatial and temporal)
    # Assuming layout: [Spatial (3x3), Temporal/Angular (4x4)]
    # Sigma_pt corresponds to the cross-terms between spatial (0-2) and temporal/angular (3-6)
    Sigma_pt = Sigma_cond[0:3, 3:7]
    
    # Calculate magnitude of spatial-temporal correlation
    correlation_magnitude = np.linalg.norm(Sigma_pt)
    
    # Check threshold relative to screen extent
    if correlation_magnitude <= 0.05 * screen_extent:
        return False
        
    # Calculate normalized temporal scale from Sigma_t (temporal block)
    Sigma_t = Sigma_cond[3:7, 3:7]
    # Assuming temporal scale is derived from the trace or max eigenvalue of Sigma_t
    # Note: Exact derivation of "normalized temporal scale" from Sigma_t is not fully specified in [S1] beyond "derived from"
    # We assume it refers to the magnitude of temporal variance components.
    temporal_scale = np.max(np.linalg.eigvals(Sigma_t))
    
    if temporal_scale > 0.25:
        return True
        
    return False

def render_conditional_gaussians(gaussians, camera_params, time_t, view_dir_d):
    """
    Renders the scene using conditional 3DGS.
    Args:
        gaussians: List of Gaussian objects with conditional parameters.
        camera_params: Camera intrinsic and extrinsic matrices.
        time_t: Current time step.
        view_dir_d: Current view direction.
    Returns:
        Image tensor.
    """
    # 1. Conditional Slicing: Convert 7DGS to 3DGS-compatible format
    # This involves evaluating the conditional dependencies for the current t and d
    projected_gaussians = []
    for g in gaussians:
        # Get conditional mean and covariance for current t, d
        mu_cond = g.get_conditional_mean(time_t, view_dir_d)
        Sigma_cond = g.get_conditional_covariance(time_t, view_dir_d)
        
        # 2. Project to 2D
        # Standard perspective projection
        mu_2d, Sigma_2d = project_to_image_plane(mu_cond, Sigma_cond, camera_params)
        
        projected_gaussians.append({
            'mu': mu_2d,
            'Sigma': Sigma_2d,
            'color': g.color,
            'opacity': g.get_conditional_opacity(time_t, view_dir_d)
        })
        
    # 3. Splatting
    # Differentiable rasterization
    image = splat_2d_gaussians(projected_gaussians, image_width, image_height)
    
    return image
```

**Edge Cases and Pitfalls:**
*   **Precision:** SVD decomposition must handle near-singular covariance matrices carefully to avoid numerical instability in scale extraction [S1].
*   **Boundaries:** The threshold for splitting ($0.05$ relative to screen extent) requires accurate calculation of screen extent, which depends on camera FOV and resolution [S1].
*   **Degenerate Input:** If temporal scale is zero (static scene), splitting criteria based on temporal cues will not trigger, relying solely on spatial gradients [S1].

## Subtopics

### [Ground Truth Downsampling](15761-ground-truth-downsampling.md)
A supervision strategy used during the low-resolution optimization stage to prevent optimization misguidance caused by aliased ground truth images. It differs from standard upsampling by explicitly downscaling target images using an anti-aliasing filter before computing the loss.

## Operating characteristics
*   **Time Complexity:** Rendering is compatible with standard 3DGS, implying $O(N)$ complexity with respect to the number of Gaussians $N$ per frame [S1].
*   **Memory Complexity:** Requires storing additional covariance blocks for temporal and angular dependencies, increasing memory per Gaussian compared to static 3DGS [S1].
*   **Measured Performance:** Evaluated on D-NeRF (800x800), Technicolor, and 7DGS-PBR datasets [S1]. Experiments conducted on NVIDIA Tesla V100 GPU (16GB) [S1].
*   **Strengths:** Integrates seamlessly with existing 3DGS training pipeline, leveraging adaptive density control and efficient rasterization with minimal modifications [S1].
*   **Limitations:** Requires careful tuning of modulation parameters ($\lambda_d, \lambda_t$) and thresholds for splitting [S1].
*   **When to Use:** Dynamic scenes with significant motion and view-dependent appearance changes [S1].
*   **When Not to Use:** Static scenes where standard 3DGS is sufficient and more efficient [S1].

## Background
Originated in the paper "7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting" [S1]. The method builds on the 3DGS framework [S1] and extends it to handle 4D scenes (3D space + time) and view-dependent effects by conditioning Gaussian parameters on time and view direction [S1]. It was evaluated on synthetic (D-NeRF) and in-the-wild (Technicolor) video datasets [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
