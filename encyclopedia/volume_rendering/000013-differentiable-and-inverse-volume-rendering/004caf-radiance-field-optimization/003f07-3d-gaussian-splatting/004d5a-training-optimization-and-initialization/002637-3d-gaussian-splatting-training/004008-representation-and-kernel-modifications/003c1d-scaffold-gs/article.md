# Scaffold-GS
Also called: Scaffold-Gaussian Splatting
## Overview
Scaffold-GS addresses the limitations of standard 3D Gaussian Splatting (3DGS) regarding initialization sparsity and the capacity to model complex view-dependent effects without excessive Gaussian proliferation. It replaces the direct attribute storage of standard 3DGS with a structured voxel grid backbone, where Gaussian attributes are derived from interpolated voxel features and lightweight neural networks. This approach distinguishes itself from standard 3DGS by enforcing structural constraints on the representation, linking it to the [Representation and Kernel Modifications](10145-representation-and-kernel-modifications.md) family, while differing from [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md) by focusing on geometric scaffolding rather than attribute sign extension.
## How it works
**Inputs and Outputs:**
*   **Input:** Sparse Structure-from-Motion (SfM) point cloud; multi-view images.
*   **Output:** A set of 3D Gaussians defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$, rendered via differentiable rasterization.

**Data Structures:**
1.  **Voxel Grid:** A 3D grid covering the scene extent. Each voxel point stores learnable features $\mathbf{f}_v$.
2.  **Gaussian Attributes:** Not stored directly. Instead, $\boldsymbol{\mu}, \boldsymbol{\Sigma}, \alpha, \mathbf{c}$ are computed on-the-fly or cached based on interpolated features.

**Method Steps:**
1.  **Voxel Grid Initialization:** Initialize a voxel grid covering the bounding box of the input SfM points. Attach learnable feature vectors $\mathbf{f}_v$ to each voxel center.
2.  **Feature Interpolation:** For a target Gaussian location $\mathbf{x}$, trilinearly interpolate the features from the surrounding voxels to obtain $\mathbf{f}_{interp}(\mathbf{x})$.
3.  **Attribute Prediction:** Use lightweight neural networks (MLPs) to map $\mathbf{f}_{interp}$ to the Gaussian attributes:
    *   $\boldsymbol{\mu}, \boldsymbol{\Sigma}$: Derived from the spatial position and potentially interpolated orientation/scale features.
    *   $\alpha, \mathbf{c}$: Predicted via MLPs taking $\mathbf{f}_{interp}$ and view direction $\mathbf{v}$ as input.
4.  **Rendering:** Render the Gaussians using standard alpha blending.
5.  **Optimization:** Backpropagate gradients through the interpolation and MLP layers to update voxel features $\mathbf{f}_v$.

**Parameters:**
*   Voxel resolution: Not given in the sources.
*   MLP depth/width: Not given in the sources.

**Edge Cases:**
*   **Out-of-bounds:** If a Gaussian center falls outside the voxel grid, features must be clamped or zero-padded. [G]
*   **Precision:** Trilinear interpolation requires careful handling of floating-point precision for gradient stability. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_gaussian_attributes(voxel_grid, gaussian_centers, view_directions):
    """
    voxel_grid: Tensor of shape [V, V, V, C] containing learnable features.
    gaussian_centers: Tensor of shape [N, 3] with Gaussian means.
    view_directions: Tensor of shape [N, 3] with view vectors.
    Returns:
    means: Tensor [N, 3]
    covariances: Tensor [N, 3, 3]
    opacities: Tensor [N, 1]
    colors: Tensor [N, 3]
    """
    # Normalize centers to voxel grid coordinates [0, 1]
    normalized_centers = normalize_to_voxel_space(gaussian_centers)
    
    # Trilinear interpolation of features
    # grid_sample requires normalized coordinates in [-1, 1]
    grid_coords = 2.0 * normalized_centers - 1.0
    grid_coords = grid_coords.unsqueeze(0).unsqueeze(0).unsqueeze(0) # [1, 1, 1, 3]
    
    # Interpolate features for all Gaussians
    # Assuming voxel_grid is [1, V, V, V, C] for grid_sample
    features = grid_sample(voxel_grid, grid_coords, mode='bilinear', padding_mode='border')
    features = features.squeeze() # [N, C]
    
    # Predict attributes using lightweight MLPs
    # Note: Standard 3DGS covariance is parameterized by rotation and scale.
    # Scaffold-GS predicts these from features.
    
    # Example MLP for opacity and color
    opacity_mlp = MLP_small()
    color_mlp = MLP_small()
    
    opacities = opacity_mlp(features)
    colors = color_mlp(features, view_directions)
    
    # Predict mean, rotation, scale from features
    # This part is simplified; actual implementation may use separate heads
    mean_offset = mean_mlp(features)
    means = gaussian_centers + mean_offset
    
    rotation = rotation_mlp(features)
    scale = scale_mlp(features)
    
    covariances = compute_covariance_from_rotation_scale(rotation, scale)
    
    return means, covariances, opacities, colors
```
## Subtopics

### [Scaffold-GS Integration](12513-scaffold-gs-integration.md)
Applies DART and AGE modules to the Scaffold-GS backbone for improved initialization and densification control.

## Operating characteristics
*   **Time Complexity:** Rendering is similar to 3DGS, but attribute prediction adds MLP inference cost per Gaussian. [S1]
*   **Memory Complexity:** Stores voxel features instead of per-Gaussian attributes. Memory usage depends on voxel resolution and feature dimension. [S1]
*   **Strengths:** Better handling of sparse regions; reduced popping artifacts; more compact representation for complex geometry. [S1]
*   **Limitations:** Requires careful tuning of voxel resolution; MLP inference can be slower than direct attribute lookup. [S1]
## Background
Scaffold-GS was proposed in [S1] as a method to improve 3DGS by initializing a voxel grid and attaching learnable features to each voxel point. All attributes of Gaussians are determined by interpolated features and lightweight neural networks. This approach was later extended by Octree-GS [S1] for level-of-detail strategies.
## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
