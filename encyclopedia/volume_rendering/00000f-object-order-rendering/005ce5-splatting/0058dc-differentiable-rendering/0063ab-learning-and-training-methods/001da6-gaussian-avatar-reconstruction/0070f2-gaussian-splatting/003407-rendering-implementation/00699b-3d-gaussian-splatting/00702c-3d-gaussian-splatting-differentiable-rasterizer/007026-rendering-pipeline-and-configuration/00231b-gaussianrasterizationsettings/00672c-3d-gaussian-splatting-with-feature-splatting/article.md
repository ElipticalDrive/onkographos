# 3D Gaussian Splatting with Feature Splatting

Also called: High Fidelity Point Cloud Rendering via Feature Splatting, PFGS (Point Cloud Gaussian Splatting with Feature Splatting)

## Overview
3D Gaussian Splatting with Feature Splatting extends the standard 3D Gaussian Splatting rendering pipeline to handle high-dimensional semantic feature tensors alongside RGB color, enabling high-fidelity point cloud rendering where both appearance and semantic information are preserved [S1, unofficial]. It differs from standard 3DGS by modifying the rasterization kernel to accumulate and differentiate multi-channel feature maps in addition to color, allowing for downstream tasks such as semantic segmentation or feature-based reconstruction [S1, unofficial]. It builds on [GaussianRasterizationSettings](2924-gaussianrasterizationsettings.md) and [Gaussian Rasterization (Forward)](17476-gaussian-rasterization-forward.md) [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `means3D`: Tensor of shape $(N, 3)$, 3D Gaussian centers [S1, unofficial].
*   `colors_precomp`: Tensor of shape $(N, 3)$, RGB colors [S1, unofficial].
*   `semantic_feature`: Tensor of shape $(N, C)$, $C$-dimensional semantic features per Gaussian [S1, unofficial].
*   `opacities`: Tensor of shape $(N, 1)$, per-Gaussian opacities [S1, unofficial].
*   `scales`: Tensor of shape $(N, 3)$, Gaussian scales [S1, unofficial].
*   `rotations`: Tensor of shape $(N, 4)$, Gaussian rotations (quaternions) [S1, unofficial].
*   `cov3Ds_precomp`: Tensor of shape $(N, 6)$, precomputed covariances (optional if scales/rotations provided) [S1, unofficial].
*   `raster_settings`: Configuration object containing camera parameters and image dimensions [S1, unofficial].

**Outputs:**
*   `color`: Rendered RGB image tensor of shape $(3, H, W)$ [S1, unofficial].
*   `feature_map`: Rendered semantic feature tensor of shape $(C, H, W)$ [S1, unofficial].
*   `radii`: Tensor of shape $(N)$, radius of each Gaussian in pixel space for visibility culling [S1, unofficial].

### Data Structures
**GaussianRasterizationSettings**
A named tuple or struct containing:
*   `image_height` (int): Height of the output image [S1, unofficial].
*   `image_width` (int): Width of the output image [S1, unofficial].
*   `tanfovx` (float): Tangent of horizontal field of view [S1, unofficial].
*   `tanfovy` (float): Tangent of vertical field of view [S1, unofficial].
*   `bg` (torch.Tensor): Background color tensor [S1, unofficial].
*   `scale_modifier` (float): Global scale modifier for Gaussian sizes [S1, unofficial].
*   `viewmatrix` (torch.Tensor): $4 \times 4$ view transformation matrix [S1, unofficial].
*   `projmatrix` (torch.Tensor): $4 \times 4$ projection matrix [S1, unofficial].
*   `sh_degree` (int): Degree of spherical harmonics (0 for precomputed colors) [S1, unofficial].
*   `campos` (torch.Tensor): Camera position [S1, unofficial].
*   `prefiltered` (bool): Whether SH coefficients are prefiltered [S1, unofficial].
*   `debug` (bool): Enable debug mode [S1, unofficial].

**CUDA Rasterizer Configuration**
The underlying CUDA kernel requires `NUM_SEMANTIC_CHANNELS` to be defined in `submodules/diff-gaussian-rasterization/cuda_rasterizer/config.h` at compile time to determine the dimensionality $C$ of the feature map [S1, unofficial].

### Numbered Steps

1.  **Input Validation and Preparation**:
    *   Check that exactly one of the covariance representations is provided: either `(scales, rotations)` or `cov3Ds_precomp` [S1, unofficial].
    *   If `scales` is `None`, set to an empty tensor [S1, unofficial].
    *   If `rotations` is `None`, set to an empty tensor [S1, unofficial].
    *   If `cov3Ds_precomp` is `None`, set to an empty tensor [S1, unofficial].
    *   Ensure `semantic_feature` is a valid tensor of shape $(N, C)$ [S1, unofficial].

2.  **Forward Pass Argument Restructuring**:
    *   The argument tuple is ordered as follows [S1, unofficial]:
        ```python
        args = (
            raster_settings.bg,
            means3D,
            colors_precomp,
            semantic_feature,
            opacities,
            scales,
            rotations,
            raster_settings.scale_modifier,
            cov3Ds_precomp,
            raster_settings.viewmatrix,
            raster_settings.projmatrix,
            raster_settings.tanfovx,
            raster_settings.tanfovy,
            raster_settings.image_height,
            raster_settings.image_width,
            sh,
            raster_settings.sh_degree,
            raster_settings.campos,
            raster_settings.prefiltered,
            raster_settings.debug
        )
        ```

3.  **Forward Rasterization Execution**:
    *   Invoke the C++/CUDA rasterizer via `_C.rasterize_gaussians(*args)` [S1, unofficial].
    *   The kernel projects 3D Gaussians to 2D, sorts them by depth, and performs alpha blending [G].
    *   The function returns `(num_rendered, color, feature_map, radii, geomBuffer, binningBuffer, imgBuffer)` [S1, unofficial].
    *   Return `(color, feature_map, radii)` [S1, unofficial].

4.  **Backward Pass Input**:
    *   The backward function receives `grad_out_color` and `grad_out_feature` (in addition to the unused `_`) [S1, unofficial].
    *   The `grad_out_feature` is the gradient of the loss with respect to the `feature_map` output [S1, unofficial].

5.  **Backward Pass Argument Restructuring**:
    *   The argument tuple is ordered as follows [S1, unofficial]:
        ```python
        args_backward = (
            raster_settings.bg,
            means3D,
            radii,
            colors_precomp,
            semantic_feature,
            scales,
            rotations,
            raster_settings.scale_modifier,
            cov3Ds_precomp,
            raster_settings.viewmatrix,
            raster_settings.projmatrix,
            raster_settings.tanfovx,
            raster_settings.tanfovy,
            grad_out_color,
            grad_out_feature,
            sh,
            raster_settings.sh_degree,
            raster_settings.campos,
            geomBuffer,
            num_rendered,
            binningBuffer,
            imgBuffer,
            raster_settings.debug
        )
        ```

6.  **Backward Rasterization Execution**:
    *   Invoke the C++/CUDA backward rasterizer via `_C.rasterize_gaussians_backward(*args_backward)` [S1, unofficial].
    *   It returns gradients for: `grad_means2D`, `grad_colors_precomp`, `grad_semantic_feature`, `grad_opacities`, `grad_means3D`, `grad_cov3Ds_precomp`, `grad_sh`, `grad_scales`, `grad_rotations` [S1, unofficial].

7.  **Backward Pass Return**:
    *   The gradients are returned in the order corresponding to the forward inputs: `(grad_means3D, grad_means2D, grad_sh, grad_colors_precomp, grad_semantic_feature, grad_opacities, grad_scales, grad_rotations, grad_cov3Ds_precomp, None)` [S1, unofficial].
    *   The last `None` corresponds to the `raster_settings` argument which does not require a gradient [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class GaussianRasterizer(nn.Module):
    def __init__(self, raster_settings):
        super().__init__()
        self.raster_settings = raster_settings
        # Import the custom CUDA extension module
        # from diff_gaussian_rasterization import _C
        self._C = None 

    def forward(self, means3D, colors_precomp, semantic_feature, opacities, 
                scales=None, rotations=None, cov3Ds_precomp=None, sh=None):
        """
        Forward pass for 3D Gaussian Splatting with Feature Splatting.
        
        Args:
            means3D: (N, 3) Tensor of 3D Gaussian centers.
            colors_precomp: (N, 3) Tensor of RGB colors.
            semantic_feature: (N, C) Tensor of semantic features.
            opacities: (N, 1) Tensor of opacities.
            scales: (N, 3) Tensor of scales (optional).
            rotations: (N, 4) Tensor of rotations (quaternions) (optional).
            cov3Ds_precomp: (N, 6) Tensor of precomputed covariances (optional).
            sh: Spherical harmonics coefficients (optional).
            
        Returns:
            color: (3, H, W) Rendered RGB image.
            feature_map: (C, H, W) Rendered semantic feature map.
            radii: (N,) Tensor of radii for visible Gaussians.
        """
        # 1. Validation
        if scales is None:
            scales = torch.empty(0)
        if rotations is None:
            rotations = torch.empty(0)
        if cov3Ds_precomp is None:
            cov3Ds_precomp = torch.empty(0)
            
        # Check that exactly one covariance representation is provided
        has_scales_rot = (len(scales) > 0 and len(rotations) > 0)
        has_cov = len(cov3Ds_precomp) > 0
        if not (has_scales_rot ^ has_cov):
            raise ValueError("Exactly one of (scales, rotations) or cov3Ds_precomp must be provided.")

        # 2. Construct Args for Forward
        rs = self.raster_settings
        args_forward = (
            rs.bg,
            means3D,
            colors_precomp,
            semantic_feature,
            opacities,
            scales,
            rotations,
            rs.scale_modifier,
            cov3Ds_precomp,
            rs.viewmatrix,
            rs.projmatrix,
            rs.tanfovx,
            rs.tanfovy,
            rs.image_height,
            rs.image_width,
            sh,
            rs.sh_degree,
            rs.campos,
            rs.prefiltered,
            rs.debug
        )

        # 3. Execute Forward Rasterization
        # _C.rasterize_gaussians returns: num_rendered, color, feature_map, radii, geomBuffer, binningBuffer, imgBuffer
        num_rendered, color, feature_map, radii, geomBuffer, binningBuffer, imgBuffer = self._C.rasterize_gaussians(*args_forward)

        # 4. Save for Backward
        self.save_for_backward(
            means3D,
            colors_precomp,
            semantic_feature,
            opacities,
            scales,
            rotations,
            cov3Ds_precomp,
            radii,
            geomBuffer,
            binningBuffer,
            imgBuffer
        )
        
        self.raster_settings_saved = rs

        return color, feature_map, radii

    def backward(self, grad_out_color, grad_out_feature):
        """
        Backward pass for 3D Gaussian Splatting with Feature Splatting.
        
        Args:
            grad_out_color: (3, H, W) Gradient of loss w.r.t. color.
            grad_out_feature: (C, H, W) Gradient of loss w.r.t. feature map.
            
        Returns:
            Gradients for all input tensors.
        """
        # Retrieve saved tensors
        means3D, colors_precomp, semantic_feature, opacities, scales, rotations, cov3Ds_precomp, radii, geomBuffer, binningBuffer, imgBuffer = self.saved_tensors
        rs = self.raster_settings_saved

        # 5. Construct Args for Backward
        args_backward = (
            rs.bg,
            means3D,
            radii,
            colors_precomp,
            semantic_feature,
            scales,
            rotations,
            rs.scale_modifier,
            cov3Ds_precomp,
            rs.viewmatrix,
            rs.projmatrix,
            rs.tanfovx,
            rs.tanfovy,
            grad_out_color,
            grad_out_feature,
            None, # sh
            rs.sh_degree,
            rs.campos,
            geomBuffer,
            None, # num_rendered
            binningBuffer,
            imgBuffer,
            rs.debug
        )
        
        # 6. Execute Backward Rasterization
        # Returns: grad_means2D, grad_colors_precomp, grad_semantic_feature, grad_opacities, grad_means3D, grad_cov3Ds_precomp, grad_sh, grad_scales, grad_rotations
        grad_means2D, grad_colors_precomp, grad_semantic_feature, grad_opacities, grad_means3D, grad_cov3Ds_precomp, grad_sh, grad_scales, grad_rotations = self._C.rasterize_gaussians_backward(*args_backward)

        # 7. Return Gradients in Forward Input Order
        return (
            grad_means3D,
            grad_means2D,
            grad_sh,
            grad_colors_precomp,
            grad_semantic_feature,
            grad_opacities,
            grad_scales,
            grad_rotations,
            grad_cov3Ds_precomp,
            None # raster_settings does not require gradient
        )
```

### Parameters and Edge Cases
*   **NUM_SEMANTIC_CHANNELS**: Must be defined at compile time in `submodules/diff-gaussian-rasterization/cuda_rasterizer/config.h` [S1, unofficial]. This determines the channel dimension $C$ of the `semantic_feature` and `feature_map` tensors [S1, unofficial].
*   **Visibility Marking**: The `markVisible` method uses `_C.mark_visible(positions, viewmatrix, projmatrix)` to return a boolean tensor of points within the camera frustum [S1, unofficial].
*   **Precision**: Standard 3DGS precision issues (e.g., depth sorting stability) apply [G]. The addition of feature channels does not change the depth sorting logic but increases memory bandwidth usage during the blending step [G].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Tested on RTX 3090 with PyTorch 1.13.0 and CUDA 11.7 [S1, unofficial]. Specific FPS numbers are not provided in the sources.
*   **Strengths**: Enables semantic-aware rendering and reconstruction from point clouds [S1, unofficial].
*   **Limitations**: Requires recompilation of the CUDA rasterizer for different feature dimensions [S1, unofficial]. Not given in the sources: specific scalability limits for very large $C$.

## Background
*   **Origin**: PFGS (Point Cloud Gaussian Splatting with Feature Splatting) was introduced by Jiaxu Wang, Ziyi Zhang, Junhao He, and Renjing Xu in 2024 [S1, unofficial].
*   **Paper**: "PFGS: High Fidelity Point Cloud Rendering via Feature Splatting," ECCV 2024 [S1, unofficial].
*   **Development**: The rasterization kernel is based on [Feature-3DGS](https://github.com/ShijieZhou-UCLA/feature-3dgs) but with minor differences and bug fixes [S1, unofficial]. It extends the original 3DGS rasterizer to support high-dimensional features [S1, unofficial].
*   **Usage**: Used for high-fidelity rendering of point clouds with semantic information, demonstrated on datasets like ScanNet, DTU, and THuman2 [S1, unofficial].

## Sources
[S1] Mercerai/PFGS, https://github.com/Mercerai/PFGS, fetched 2026-10-28 (unofficial)
