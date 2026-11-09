# Conditional 3DGS Optimization and Rendering

Also called: 7DGS to Conditional 3DGS, Optimization and Rendering Pipeline

## Overview
This method adapts the 3D Gaussian Splatting (3DGS) training and rendering pipeline to handle 7-dimensional data (3D spatial, 1D temporal, 3D directional) by converting conditional representations into standard 3DGS-compatible formats for rasterization [S1]. It distinguishes itself from sibling methods by leveraging temporal cues for adaptive density control (splitting) and modifying pruning thresholds to account for time and view-direction modulation, allowing seamless integration with existing 3DGS infrastructure [S1].

## How it works
**Inputs:** Conditional covariance matrix $\Sigma_{cond}$ (derived from conditioning on time $t$ and view direction $d$), spatial-temporal correlation block $\Sigma_{pt}$, temporal scale $\Sigma_t$, and standard 3DGS scene parameters (means, opacities, colors) [S1].
**Outputs:** Projected 2D Gaussians for rasterization and updated Gaussian attributes (split/pruned) based on conditional metrics [S1].

### Data Structures
The method operates on the conditional covariance matrix $\Sigma_{cond}$, which is a $7 \times 7$ symmetric positive-definite matrix representing the joint distribution of spatial, temporal, and angular dimensions [S1]. It is decomposed into orthogonal matrix $U$ and diagonal matrix $D$ [S1].

### Algorithm Steps

1.  **Extract Scale and Rotation:**
    Perform Singular Value Decomposition (SVD) on the conditional covariance matrix:
    $$ \Sigma_{cond} = U D U^T $$
    where $U$ is an orthogonal matrix and $D$ is a diagonal matrix containing singular values [S1].
    Define the rotation matrix $R$ as:
    $$ R = U $$
    Compute the scale vector $S$ as the diagonal elements of $D$:
    $$ S = \text{diag}(D) $$
    Ensure $R$ represents a right-handed coordinate system by adjusting its last column:
    $$ R_{:,3} = R_{:,3} \cdot \text{sign}(\det(R)) $$
    [S1]

2.  **Conditional Gaussian Splitting:**
    Quantify spatial-temporal correlation using the magnitude of the off-diagonal block $\Sigma_{pt}$, which captures interaction between spatial and temporal components [S1].
    Split a Gaussian if both conditions are met:
    *   The magnitude of $\Sigma_{pt}$ exceeds a threshold of $0.05$ (relative to screen extent) [S1].
    *   The normalized temporal scale (derived from $\Sigma_t$) is larger than $0.25$ [S1].
    This criterion ensures regions with significant motion dynamics are densely represented [S1].

3.  **Rendering Pipeline:**
    *   **Conditional Slicing:** Convert the 7DGS representation into a 3DGS-compatible format via conditional slicing [S1].
    *   **Projection:** Project the resulting conditional 3D Gaussian mean and covariance onto the image plane using standard perspective projection to yield 2D Gaussians [S1].
    *   **Rasterization:** Splat the 2D Gaussians onto the image canvas using a differentiable rasterization routine [S1].
    *   **Compositing:** Compute final pixel colors by aggregating contributions in a depth-aware, opacity-blended manner [S1].

4.  **Pruning and Optimization:**
    Use the standard 3DGS loss functions and optimizers [S1]. Increase the minimum opacity threshold for pruning to $\tau_{min} = 0.01$ to compensate for the modulation of conditional opacity $\alpha_{cond}$ by time and view direction [S1].

### Parameters
*   **Splitting Thresholds:** Spatial-temporal correlation threshold = $0.05$; Normalized temporal scale threshold = $0.25$ [S1].
*   **Pruning Threshold:** $\tau_{min} = 0.01$ [S1].
*   **Modulation Parameters:** Directional ($\lambda_d$) and temporal ($\lambda_t$) modulation parameters are initially set to:
    *   $0.5$ for 7DGS-PBR
    *   $0.05$ for D-NeRF
    *   $0.1$ for Technicolor
    These parameters become trainable after 15,000 iterations [S1].
*   **Initialization:** Scenes (except Technicolor) use 100,000 randomly initialized points within a bounding cube; Technicolor uses COLMAP sparse reconstructions; specific scenes (heart1, heart2) use marching cubes following DDGS [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_scale_rotation(sigma_cond):
    """
    Extracts scale and rotation from conditional covariance matrix.
    Args:
        sigma_cond: 7x7 conditional covariance matrix.
    Returns:
        R: 7x7 rotation matrix (right-handed).
        S: 7x1 scale vector.
    """
    # SVD Decomposition
    U, D, _ = svd(sigma_cond)
    
    # Define rotation and scale
    R = U
    S = diag(D)
    
    # Ensure right-handed coordinate system
    det_R = det(R)
    if det_R < 0:
        R[:, 3] = -R[:, 3]
        
    return R, S

def check_splitting_criteria(sigma_pt, sigma_t, screen_extent):
    """
    Determines if a Gaussian should be split based on temporal cues.
    Args:
        sigma_pt: Off-diagonal block capturing spatial-temporal correlation.
        sigma_t: Temporal scale component.
        screen_extent: Normalization factor for screen extent.
    Returns:
        bool: True if splitting criteria are met.
    """
    # Quantify spatial-temporal correlation
    correlation_magnitude = abs(sigma_pt)
    
    # Thresholds
    corr_threshold = 0.05 * screen_extent
    temp_scale_threshold = 0.25
    
    # Normalize temporal scale (derived from sigma_t)
    # Note: Exact normalization formula for sigma_t is not specified in sources [G]
    normalized_temp_scale = normalize_temporal_scale(sigma_t) 
    
    # Check conditions
    if correlation_magnitude > corr_threshold and normalized_temp_scale > temp_scale_threshold:
        return True
    return False

def render_conditional_gaussians(gaussians, camera_params):
    """
    Renders conditional 3D Gaussians using standard 3DGS pipeline.
    Args:
        gaussians: List of conditional 3D Gaussians (converted from 7DGS).
        camera_params: Camera intrinsic and extrinsic matrices.
    Returns:
        image: Rendered image tensor.
    """
    # Conditional Slicing (Implementation details in Section 4.2 of [S1])
    # Convert 7DGS to 3DGS-compatible format
    conditional_3d_gaussians = conditional_slice(gaussians)
    
    # Project to 2D
    projected_gaussians = []
    for g in conditional_3d_gaussians:
        mean_2d, cov_2d = perspective_project(g.mean, g.cov, camera_params)
        projected_gaussians.append({'mean': mean_2d, 'cov': cov_2d, 'opacity': g.opacity, 'color': g.color})
        
    # Differentiable Rasterization
    image = differentiable_rasterize(projected_gaussians, camera_params.image_shape)
    
    return image

def update_pruning_threshold(current_opacity, tau_min=0.01):
    """
    Applies pruning based on modified opacity threshold.
    Args:
        current_opacity: Current opacity of the Gaussian.
        tau_min: Minimum opacity threshold (increased to 0.01).
    Returns:
        bool: True if Gaussian should be pruned.
    """
    return current_opacity < tau_min
```

## Operating characteristics
*   **Compatibility:** Fully compatible with existing 3DGS training pipelines, loss functions, and optimizers [S1].
*   **Performance:** Evaluated on NVIDIA Tesla V100 GPU (16GB memory) [S1]. Batch size of 4 with downscaling factor of 2 (except Technicolor, which uses batch size 1 and no downscaling) [S1].
*   **Strengths:** Integrates seamlessly with 3DGS infrastructure, leveraging adaptive density control and efficient rasterization with minimal modifications [S1].
*   **Limitations:** Requires specific initialization strategies for different datasets (random points, COLMAP, marching cubes) [S1].

## Background
This method is part of the 7DGS framework, which unifies spatial, temporal, and angular Gaussian splatting [S1]. It builds on the 3DGS training pipeline, modifying it to handle conditional data by adjusting pruning thresholds and leveraging temporal cues for densification [S1]. The approach was introduced in the paper "7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting" [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
