# MIP Scale Computation

Also called: Gaussian Scale Factor Computation, MIP Scale

## Overview
Computes a per-Gaussian scale factor based on the Euclidean distance from the camera center and the camera's focal length, primarily used for anti-aliasing or level-of-detail (LOD) adjustments in Gaussian splatting [S1, unofficial]. It differs from standard projected voxel size calculations by operating on parametric Gaussian primitives rather than volumetric voxels, scaling the representation size inversely to the focal length to maintain consistent screen-space coverage [S1, unofficial].

## How it works
**Inputs:**
- `viewpoint_camera`: Object containing `camera_center` (3D vector), `FoVx` (horizontal field of view in radians or degrees, depending on implementation), and `image_width` (pixel width).
- `pc`: Gaussian model object containing `get_xyz` (tensor of shape $[N, 3]$ representing Gaussian centers).

**Outputs:**
- `scales`: Tensor of shape $[N]$ containing the computed scale factor for each Gaussian.

**Data Structures:**
- `dist_pp`: Tensor of shape $[N, 1]$ storing Euclidean distances.
- `focal`: Scalar value representing the focal length in pixels.

**Steps:**
1. Calculate the Euclidean distance from the camera center to each Gaussian center.
   $$ \text{dist\_pp}_i = \| \mathbf{p}_i - \mathbf{c} \| $$
   where $\mathbf{p}_i$ is the $i$-th Gaussian center and $\mathbf{c}$ is the camera center.
2. Convert the horizontal field of view ($\text{FoV}_x$) and image width ($W$) to focal length ($f$). The standard conversion is:
   $$ f = \frac{W}{2 \tan(\frac{\text{FoV}_x}{2})} $$
   [S1, unofficial]
3. Compute the scale factor by dividing the distance by the focal length:
   $$ s_i = \frac{\text{dist\_pp}_i}{f} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def fov2focal(fov, width):
    """Convert horizontal field of view to focal length in pixels."""
    # Assuming fov is in radians. If degrees, convert first.
    return width / (2 * math.tan(fov / 2))

def mip_scales(viewpoint_camera, pc):
    """
    Compute MIP scales for Gaussian splatting.
    
    Args:
        viewpoint_camera: Object with attributes camera_center, FoVx, image_width.
        pc: GaussianModel object with attribute get_xyz (tensor [N, 3]).
        
    Returns:
        scales: Tensor [N] of scale factors.
    """
    # 1. Get Gaussian centers
    xyz = pc.get_xyz  # Shape: [N, 3]
    
    # 2. Get camera center and broadcast to match N
    camera_center = viewpoint_camera.camera_center  # Shape: [3]
    # Repeat camera_center to match batch size if necessary, though norm handles broadcasting
    # Explicit repeat for clarity in some frameworks
    camera_center_expanded = camera_center.repeat(xyz.shape[0], 1)
    
    # 3. Calculate Euclidean distance
    # dist_pp shape: [N, 1]
    dist_pp = torch.norm(xyz - camera_center_expanded, dim=1, keepdim=True)
    
    # 4. Calculate focal length
    # fov2focal implementation depends on input units of FoVx
    focal = fov2focal(viewpoint_camera.FoVx, viewpoint_camera.image_width)
    
    # 5. Return ratio
    # dist_pp is [N, 1], focal is scalar. Result is [N, 1] or squeezed to [N]
    scales = dist_pp / focal
    
    return scales.squeeze(-1)
```

**Edge Cases and Pitfalls:**
- **Zero Distance:** If a Gaussian center coincides with the camera center, the distance is zero, resulting in a scale of zero. This may cause division issues if not handled or result in invisible splats. [G] Guard against `dist_pp < epsilon`.
- **FOV Units:** The `fov2focal` function assumes radians. If `FoVx` is in degrees, it must be converted (`fov * pi / 180`). [G] Check source implementation details.
- **Focal Length Zero:** If `FoVx` is 0, `tan(0)` is 0, leading to division by zero. [G] Clamp `FoVx` to a minimum non-zero value.

## Subtopics

### Mip-Splatting Renderer V2
This variant retrieves pre-filtered 3D scales and opacities directly from the Gaussian model via `get_3d_filtered_scales_and_opacities()` rather than computing scale from camera distance alone [S2, unofficial]. It returns the squeezed scales and opacities. It introduces a parameter `filter_2d_kernel_size` (default 0.1) which likely influences the filtering process applied to the scales [S2, unofficial]. Unlike the standard method, it may ignore standard opacity calculation in certain status returns, returning `None` for opacity in specific paths [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of Gaussians. The operation involves a single pass over all Gaussian centers to compute distances and a constant-time focal length calculation.
- **Memory Complexity:** $O(N)$ to store the distance tensor `dist_pp` before squeezing.
- **Strengths:** Simple, differentiable (if implemented with PyTorch/TensorFlow ops), and efficient for real-time rendering. It provides a geometric basis for LOD without requiring complex spatial queries.
- **Limitations:** Relies on accurate camera parameters. Does not account for Gaussian orientation or anisotropy directly, only distance. May not be sufficient for high-quality anti-aliasing in complex scenes without additional filtering.

## Background
The method originates from the Gaussian Splatting literature, specifically implementations dealing with multi-resolution pyramid (MIP) rendering to reduce aliasing [S1, unofficial]. It is a core component in pipelines like DNGaussian and Gaussian Splatting Lightning, where scale adaptation is crucial for visual quality at varying distances [S1, unofficial][S2, unofficial].

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
[S2] yzslab/gaussian-splatting-lightning, https://github.com/yzslab/gaussian-splatting-lightning, fetched 2026-10-27 (unofficial)
