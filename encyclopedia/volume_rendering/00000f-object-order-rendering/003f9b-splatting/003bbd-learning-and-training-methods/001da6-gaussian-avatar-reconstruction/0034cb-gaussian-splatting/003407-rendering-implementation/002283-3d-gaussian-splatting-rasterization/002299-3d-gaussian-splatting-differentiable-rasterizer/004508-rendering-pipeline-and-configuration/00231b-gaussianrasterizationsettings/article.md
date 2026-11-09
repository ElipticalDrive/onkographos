# GaussianRasterizationSettings

Also called: GaussianRasterizationSettings configuration

## Overview
`GaussianRasterizationSettings` is a configuration data structure that encapsulates the camera parameters, background color, and rasterization flags required by the 3D Gaussian Splatting differentiable rasterizer. It serves as the bridge between the scene/camera state and the CUDA kernel, ensuring that the projection from 3D world space to 2D screen space is computed correctly for alpha compositing. It is distinct from the Gaussian primitives themselves (position, scale, rotation, opacity, SH coefficients) and the rasterization engine logic; it provides the static or per-frame context needed to transform and render those primitives.

## How it works
The settings object is instantiated per view (camera pose) and passed to the rasterization kernel. It contains pre-computed projection constants to avoid redundant calculations inside the tight rendering loop.

### Inputs and Outputs
*   **Inputs**: Camera intrinsic parameters (FOV, principal point), extrinsic parameters (view matrix, projection matrix), background color tensor, and boolean flags for specific rasterization behaviors.
*   **Outputs**: None directly; it configures the state for the `GaussianRasterizer` forward and backward passes.

### Data Structures
The settings are typically implemented as a `NamedTuple` or a lightweight struct. The fields are:

1.  `image_height` (int): Height of the output image in pixels.
2.  `image_width` (int): Width of the output image in pixels.
3.  `tanfovx` (float): Tangent of half the horizontal field of view.
4.  `tanfovy` (float): Tangent of half the vertical field of view.
5.  `cx` (float): Principal point x-coordinate in pixels.
6.  `cy` (float): Principal point y-coordinate in pixels.
7.  `bg` (torch.Tensor): Background color tensor, shape `(3,)` or `(4,)`, dtype float32, on the same device as the Gaussians.
8.  `scale_modifier` (float): Multiplicative scalar applied to Gaussian scales during rasterization. Default: `1.0`.
9.  `viewmatrix` (torch.Tensor): 4x4 view matrix (world-to-camera transformation).
10. `projmatrix` (torch.Tensor): 4x4 projection matrix (camera-to-clip space).
11. `sh_degree` (int): The spherical harmonic degree used for color representation.
12. `campos` (torch.Tensor): Camera position in world space, shape `(3,)`.
13. `prefiltered` (bool): Flag indicating if the SH coefficients are prefiltered for the current view direction.
14. `backward_geometry` (bool): Flag passed to the CUDA backend to enable geometry gradient computation.
15. `computer_pseudo_normal` (bool): Flag to enable pseudo-normal computation in the rasterizer.
16. `debug` (bool): Flag to enable debug outputs.

*Note: Some implementations may include additional fields such as `return_depth` to control whether depth buffers are returned, though this is often handled by the rasterizer wrapper rather than the core settings struct itself.*

### Formulas and Computation
The tangent FOV values are computed from the camera's field of view angles:

$$
\tan\text{fovx} = \tan\left(\frac{\text{FoV}_x}{2}\right)
$$

$$
\tan\text{fovy} = \tan\left(\frac{\text{FoV}_y}{2}\right)
$$

The principal point coordinates are extracted from the intrinsic matrix $K$:

$$
c_x = K_{0,2}
$$

$$
c_y = K_{1,2}
$$

### Parameters and Typical Values
*   `scale_modifier`: Typically `1.0`. Used to scale the Gaussian covariance during rasterization, often for depth-of-field effects or scale regularization.
*   `sh_degree`: Matches the training configuration, typically `2` (9 coefficients per color channel).
*   `bg`: Usually a zero tensor `(0.0, 0.0, 0.0)` for synthetic scenes or a sampled background color for real-world scenes.

### Edge Cases and Pitfalls
*   **Precision**: The `viewmatrix` and `projmatrix` must be in the correct coordinate system (typically right-handed for OpenGL-style or left-handed for DirectX-style, depending on the rasterizer implementation). Mismatched coordinate systems cause inverted or skewed splats.
*   **Device Mismatch**: `bg`, `viewmatrix`, `projmatrix`, and `campos` must reside on the same CUDA device as the Gaussian attributes.
*   **Zero Division**: If `tanfovx` or `tanfovy` are zero (zero FOV), the projection is undefined. This is a degenerate input case.

### Pseudocode
The following Python-like pseudocode demonstrates the creation of `GaussianRasterizationSettings` from a camera object.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def create_gaussian_rasterization_settings(
    image_height: int,
    image_width: int,
    camera,
    bg_color: torch.Tensor,
    scale_modifier: float = 1.0,
    sh_degree: int = 2,
    backward_geometry: bool = False,
    computer_pseudo_normal: bool = False,
    debug: bool = False
) -> dict:
    """
    Constructs the GaussianRasterizationSettings configuration.
    
    Args:
        image_height: Height of the render target.
        image_width: Width of the render target.
        camera: Object containing FoVx, FoVy, world_view_transform, full_proj_transform, 
                intrinsic matrix, and position.
        bg_color: Background color tensor (3 or 4 channels).
        scale_modifier: Multiplier for Gaussian scales.
        sh_degree: Spherical harmonic degree.
        backward_geometry: Enable geometry gradients.
        computer_pseudo_normal: Enable pseudo-normal computation.
        debug: Enable debug mode.
        
    Returns:
        A dictionary or NamedTuple representing the settings.
    """
    # 1. Compute tangent half-FOV
    # Source [S1, S4, unofficial]
    tanfovx = math.tan(camera.FoVx * 0.5)
    tanfovy = math.tan(camera.FoVy * 0.5)
    
    # 2. Extract principal point from intrinsic matrix
    # Source [S1, unofficial]
    # Assuming camera.intrinsic is a 3x3 matrix
    cx = camera.intrinsic[0, 2]
    cy = camera.intrinsic[1, 2]
    
    # 3. Extract view and projection matrices
    # Source [S1, unofficial]
    viewmatrix = camera.world_view_transform
    projmatrix = camera.full_proj_transform
    
    # 4. Extract camera position
    # Source [S1, unofficial]
    campos = camera.get_camera_position()
    
    # 5. Construct the settings object
    # Using a dictionary for clarity, though NamedTuple is common in implementation
    settings = {
        "image_height": int(image_height),
        "image_width": int(image_width),
        "tanfovx": float(tanfovx),
        "tanfovy": float(tanfovy),
        "cx": float(cx),
        "cy": float(cy),
        "bg": bg_color,
        "scale_modifier": float(scale_modifier),
        "viewmatrix": viewmatrix,
        "projmatrix": projmatrix,
        "sh_degree": int(sh_degree),
        "campos": campos,
        "prefiltered": False, # Default; set to True if SHs are prefiltered
        "backward_geometry": bool(backward_geometry),
        "computer_pseudo_normal": bool(computer_pseudo_normal),
        "debug": bool(debug)
    }
    
    return settings
```

## Subtopics

### 3D Half-Gaussian Rasterization
A variant where each 3D Gaussian is split into two halves by a plane through its center, allowing distinct opacity values ($\alpha_1, \alpha_2$) for each half to better represent sharp edges and planar surfaces [S2]. The splitting plane normal $\mathbf{n}$ is learned and stored in the three unused normal parameters of the original 3D-GS kernel [S2]. This increases the parameter count by only one additional opacity term per Gaussian compared to standard 3D-GS [S2]. It is chosen when accurate representation of planar surfaces and sharp edges is critical, such as in architectural scenes [S2].

### Restricting Gaussian Kernel Support
A technique to reduce Gaussian footprints and superfluous ray-Gaussian intersections using a generalized kernel exponent and adaptive alpha clamping [S3]. The generalized kernel is defined as $\exp \left[ -\frac{1}{2n_{\text{expo}}} ((x - \mu)^T \Sigma^{-1} (x - \mu))^{n_{\text{expo}}} \right]$, where $n_{\text{expo}}$ controls bulkiness [S3]. Pitfalls include the choice of denominator: 3DGRT uses $4.5 n_{\text{expo}}$ while GRay uses $1 2^{n_{\text{expo}}}$ to make initialization less sensitive and result in smaller initial Gaussians [S3]. Dynamic clamping can cause visible artifacts for large Gaussians, though this is less apparent in differentiable rendering (DI) where Gaussians tend to stay small [S3].

## Operating characteristics
*   **Time Complexity**: The creation of `GaussianRasterizationSettings` is $O(1)$ relative to the number of Gaussians, as it depends only on camera parameters. The actual rasterization cost is $O(N)$ where $N$ is the number of Gaussians, but the settings object itself does not contribute to this complexity.
*   **Memory Complexity**: $O(1)$ fixed size, containing a few floats, two 4x4 matrices, and a small tensor for background color.
*   **Strengths**: Pre-computing tangent FOV and principal point reduces per-pixel/per-splat computation in the CUDA kernel.
*   **Limitations**: Must be recreated or updated for every change in camera pose or intrinsic parameters.
*   **When to use**: Every frame in a differentiable rendering pipeline where camera parameters change.

## Background
The `GaussianRasterizationSettings` structure is part of the 3D Gaussian Splatting (3D-GS) rendering implementation, originally introduced by Kerbl et al. in 2023 [S1, S4, unofficial]. It was designed to efficiently pass camera state to the CUDA-based differentiable rasterizer, enabling real-time rendering of millions of Gaussians. The specific field names and structure have been adopted and adapted in various open-source implementations and extensions of 3D-GS.

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] 3D-HGS: 3D Half-Gaussian Splatting*, https://doi.org/10.1109/cvpr52734.2025.01027, fetched 2026-10-30
[S3] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
[S4] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
