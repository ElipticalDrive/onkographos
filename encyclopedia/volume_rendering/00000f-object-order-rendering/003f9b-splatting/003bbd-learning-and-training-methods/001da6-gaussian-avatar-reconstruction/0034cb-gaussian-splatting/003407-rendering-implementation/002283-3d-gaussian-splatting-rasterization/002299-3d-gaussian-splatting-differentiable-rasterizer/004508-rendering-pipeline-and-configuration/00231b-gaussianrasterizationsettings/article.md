# GaussianRasterizationSettings

Also called: GaussianRasterizationSettings configuration

## Overview
`GaussianRasterizationSettings` is a configuration object that bundles camera intrinsics, extrinsics, and rasterization flags into a single structure passed to the 3D Gaussian Splatting CUDA kernel [Parent]. It solves the problem of managing disparate rendering parameters by encapsulating them to avoid passing individual arguments to the backend [Parent]. It sets apart from sibling configurations by enforcing mutually exclusive representations for color (Spherical Harmonics vs. pre-computed RGB) and geometry (parametric scales/rotations vs. pre-computed covariances) while handling empty tensor fallbacks for missing inputs [Parent].

## How it works
**Inputs and Outputs:**
The object is instantiated with camera parameters and rendering flags. It is consumed by the `GaussianRasterizer` CUDA kernel to transform 3D Gaussians into 2D screen-space splats.

**Data Structures:**
The settings are defined as a `NamedTuple` with the following fields [S1, unofficial]:
- `image_height` (int): Height of the output image in pixels.
- `image_width` (int): Width of the output image in pixels.
- `tanfovx` (float): Tangent of half the horizontal field of view.
- `tanfovy` (float): Tangent of half the vertical field of view.
- `cx` (float): Principal point x-coordinate in pixels.
- `cy` (float): Principal point y-coordinate in pixels.
- `bg` (torch.Tensor): Background color tensor, shape `(3,)` or `(N, 3)`, residing on the CUDA device [Parent].
- `scale_modifier` (float): Multiplicative scalar applied to Gaussian scales. Default is 1.0 [S1, S4, unofficial].
- `viewmatrix` (torch.Tensor): 4x4 view transformation matrix, residing on the CUDA device [Parent].
- `projmatrix` (torch.Tensor): 4x4 projection matrix, residing on the CUDA device [Parent].
- `sh_degree` (int): Degree of Spherical Harmonics used for color representation.
- `campos` (torch.Tensor): Camera position vector, residing on the CUDA device [Parent].
- `prefiltered` (bool): Flag indicating if input Gaussians are pre-filtered.
- `backward_geometry` (bool): Flag to enable geometry gradient computation in the backend [S1, unofficial].
- `computer_pseudo_normal` (bool): Flag to enable pseudo-normal computation [S1, unofficial].
- `debug` (bool): Flag to enable debug mode.

**Computation of Intrinsics:**
The tangent field-of-view values and principal points are derived from the camera object [S1, S4, unofficial]:
1.  Compute `tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)`.
2.  Compute `tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)`.
3.  Extract `cx = intrinsic[0, 2]`.
4.  Extract `cy = intrinsic[1, 2]`.

**Pseudocode:**
The following Python-like pseudocode demonstrates the initialization of `GaussianRasterizationSettings` and the setup of auxiliary tensors required for the rasterization pipeline [S4, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def setup_gaussian_rasterization_settings(viewpoint_camera, pc, image_height, image_width):
    """
    Initialize GaussianRasterizationSettings and auxiliary tensors for 3D Gaussian Splatting.
    
    Args:
        viewpoint_camera: Object containing FoVx, FoVy, world_view_transform, and intrinsic matrix.
        pc: PointCloud object containing Gaussian attributes (xyz, features, etc.).
        image_height: int, height of the render.
        image_width: int, width of the render.
        
    Returns:
        settings: GaussianRasterizationSettings instance.
        screenspace_points: Tensor for storing screen-space points and gradients.
    """
    # 1. Compute intrinsic parameters
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    
    # Extract principal point from intrinsic matrix
    # Note: Assuming viewpoint_camera.intrinsic is a torch.Tensor
    cx = viewpoint_camera.intrinsic[0, 2]
    cy = viewpoint_camera.intrinsic[1, 2]
    
    # 2. Prepare background color
    # bg must be on the same device as Gaussian data [Parent]
    bg = torch.zeros(3, dtype=torch.float32, device=pc.get_xyz.device)
    
    # 3. Initialize GaussianRasterizationSettings
    settings = GaussianRasterizationSettings(
        image_height=int(image_height),
        image_width=int(image_width),
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        cx=float(cx),
        cy=float(cy),
        bg=bg,
        scale_modifier=1.0, # Default scale modifier [S1, S4, unofficial]
        viewmatrix=viewpoint_camera.world_view_transform,
        projmatrix=viewpoint_camera.full_proj_transform, # Assuming full projection is needed
        sh_degree=pc.get_sh_degree(), # Assuming pc exposes sh_degree
        campos=viewpoint_camera.camera_center, # Camera position
        prefiltered=False,
        backward_geometry=True, # Enable geometry gradients [S1, unofficial]
        computer_pseudo_normal=False, # Default state
        debug=False
    )
    
    # 4. Create screenspace_points tensor for gradient retention
    # This tensor stores the 2D positions of Gaussians in screen space
    # It must have the same dtype and device as pc.get_xyz [S4, unofficial]
    screenspace_points = torch.zeros_like(pc.get_xyz, dtype=pc.get_xyz.dtype, requires_grad=True, device=pc.get_xyz.device) + 0
    
    try:
        screenspace_points.retain_grad()
    except AttributeError:
        # Fallback for older PyTorch versions where retain_grad might not be available or behaves differently
        pass
        
    return settings, screenspace_points
```

**Edge Cases and Pitfalls:**
-   **Device Consistency:** All configuration tensors (`bg`, `viewmatrix`, `projmatrix`, `campos`) must reside on the same CUDA device as the Gaussian data. Mismatched devices cause kernel launch errors [Parent].
-   **Mutually Exclusive Representations:** The system enforces that either Spherical Harmonics (`shs`) or pre-computed RGB (`colors_precomp`) are provided, but not both. Similarly, either parametric scales/rotations (`scales`, `rotations`) or pre-computed covariances (`cov3D_precomp`) are used [Parent].
-   **Empty Tensors:** Missing inputs are replaced with empty tensors of shape `(N, 0)` rather than raising errors, provided a valid alternative representation exists [Parent].

## Subtopics

### 3D Half-Gaussian Rasterization
A rasterization technique that represents a 3D Gaussian kernel as two halves split by a plane through its center, allowing distinct opacity values for each half to better represent sharp edges and planar surfaces [S2]. It sets apart from standard 3D-GS by introducing a learned splitting plane normal and two opacity parameters ($\alpha_1, \alpha_2$) [S2]. Choose this when rendering scenes with sharp planar features or edges where standard isotropic Gaussians blur details [S2]. It utilizes the three unused normal parameters in the original 3D-GS kernel implementation to store the splitting plane normal vector $\mathbf{n} = [n_1, n_2, n_3]^T$ [S2]. The opacity $\alpha_1$ applies where $\mathbf{n}^T(\mathbf{x}-\boldsymbol{\mu}) \ge 0$, and $\alpha_2$ applies where $\mathbf{n}^T(\mathbf{x}-\boldsymbol{\mu}) < 0$ [S2].

### Restricting Gaussian Kernel Support
A technique to reduce Gaussian footprints and superfluous ray-Gaussian intersections using a generalized kernel exponent and adaptive alpha clamping [S3]. It sets apart from standard splatting by modifying the kernel decay function to control the "bulkiness" of the splats [S3]. Choose this when optimizing for rendering speed or reducing overdraw in dense scenes [S3]. The generalized Gaussian kernel is defined as:
$$ \exp \left[ -\frac{1}{2n_{\text{expo}}} ((x - \mu)^T \Sigma^{-1} (x - \mu))^{n_{\text{expo}}} \right] $$
where $n_{\text{expo}}$ is the exponent controlling bulkiness [S3]. A pitfall is the discrepancy in denominator constants: 3DGRT uses $4.5 \cdot 2^{n_{\text{expo}}}$ while GRay uses $1 \cdot 2^{n_{\text{expo}}}$ to make initialization less sensitive and result in smaller initial Gaussians [S3]. Dynamic clamping can cause visible artifacts for large Gaussians, though this is less apparent when most Gaussians remain small [S3].

### Gaussian Rasterization Configuration Setup
This variant emphasizes the specific setup steps for the rendering pipeline, including the creation of `screenspace_points` for gradient retention [S4, unofficial]. It sets apart by explicitly handling PyTorch version compatibility for `retain_grad()` and including a `return_depth` flag in some implementations [S4, unofficial]. Choose this when implementing the full training loop that requires backpropagation through the screen-space positions [S4, unofficial]. The `scaling_modifier` is explicitly passed as a parameter with a default of 1.0 [S4, unofficial].

## Operating characteristics
-   **Time Complexity:** The configuration object itself has $O(1)$ creation time. The cost is incurred during the rasterization kernel launch, which depends on the number of Gaussians and image resolution, not the settings object size.
-   **Memory Complexity:** The settings object occupies minimal memory (a few floats and tensors). The dominant memory usage comes from the Gaussian data and the `screenspace_points` tensor, which is $O(N)$ where $N$ is the number of Gaussians [S4, unofficial].
-   **Strengths:** Centralized parameter management reduces API complexity and ensures consistency between camera and rasterization parameters [Parent]. Explicit handling of device placement prevents runtime errors [Parent].
-   **Limitations:** Requires careful synchronization of tensor devices [Parent]. The `backward_geometry` flag adds computational overhead if gradients for geometry are not needed [S1, unofficial].
-   **When to use:** Always use this object when invoking the 3D Gaussian Splatting rasterizer to ensure correct camera projection and rendering behavior.

## Background
The `GaussianRasterizationSettings` structure is part of the 3D Gaussian Splatting rendering implementation, originating from the work on differentiable rasterization of 3D Gaussians [Parent]. It was developed to facilitate efficient training and rendering of Gaussian splats by encapsulating the necessary camera and rendering context [Parent]. The specific fields and behavior are documented in unofficial implementations and repositories such as VEG and AVM-GS [S1, S4, unofficial].

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] 3D-HGS: 3D Half-Gaussian Splatting*, https://doi.org/10.1109/cvpr52734.2025.01027, fetched 2026-10-30
[S3] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
[S4] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
