# GaussianRasterizationSettings

Also called: GaussianRasterizationSettings configuration

## Overview
`GaussianRasterizationSettings` is a configuration structure that bundles camera intrinsics, extrinsics, and rasterization flags into a single object passed to the 3D Gaussian Splatting differentiable rasterizer. It solves the problem of passing per-frame rendering context (such as field of view, view/projection matrices, and background color) to the CUDA kernel without relying on global state. It sets itself apart from sibling configuration objects by explicitly defining the geometric projection parameters (`tanfovx`, `tanfovy`, `cx`, `cy`) and control flags (`backward_geometry`, `scale_modifier`) that dictate how the splatting process transforms 3D Gaussians into 2D screen-space contributions.

## How it works
The settings object is typically implemented as a `NamedTuple` or a similar immutable data structure in Python, which is then unpacked or passed to the CUDA extension. The rasterizer uses these parameters to transform 3D Gaussian centers into camera space, project them to screen space, and determine the 2D covariance matrix for splatting.

### Inputs and Outputs
*   **Inputs:** Camera parameters (FOV, principal point, view/projection matrices), scene background color, and rasterization control flags.
*   **Outputs:** The structure itself is an input to the rasterizer kernel. The rasterizer outputs the rendered image, depth, and potentially gradients for the Gaussian attributes.

### Data Structures
The `GaussianRasterizationSettings` is defined with the following fields [S1, S4, unofficial]:

| Field | Type | Description |
| :--- | :--- | :--- |
| `image_height` | `int` | Height of the output image in pixels. |
| `image_width` | `int` | Width of the output image in pixels. |
| `tanfovx` | `float` | Tangent of half horizontal field of view. |
| `tanfovy` | `float` | Tangent of half vertical field of view. |
| `cx` | `float` | Principal point x-coordinate in pixels. |
| `cy` | `float` | Principal point y-coordinate in pixels. |
| `bg` | `torch.Tensor` | Background color tensor (typically shape `(3,)` or `(N, 3)`). |
| `scale_modifier` | `float` | Multiplicative scalar applied to Gaussian scales. |
| `viewmatrix` | `torch.Tensor` | Camera view matrix (world-to-camera transformation). |
| `projmatrix` | `torch.Tensor` | Camera projection matrix. |
| `sh_degree` | `int` | Degree of spherical harmonics used for color. |
| `campos` | `torch.Tensor` | Camera position in world coordinates. |
| `prefiltered` | `bool` | Flag indicating if SH coefficients are prefiltered. |
| `backward_geometry` | `bool` | Flag to enable geometry gradient computation. |
| `computer_pseudo_normal` | `bool` | Flag to enable pseudo-normal computation. |
| `debug` | `bool` | Flag to enable debug mode. |
| `return_depth` | `bool` | Flag to return depth buffer alongside color. |

*Note: The field `computer_pseudo_normal` appears in the source as a typo for `compute_pseudo_normal` [S1, unofficial].*

### Computation of Parameters
The parameters are computed from the camera object as follows [S1, S4, unofficial]:

1.  **Field of View Tangents:**
    $$ \text{tanfovx} = \tan\left(\frac{\text{FoV}_x}{2}\right) $$
    $$ \text{tanfovy} = \tan\left(\frac{\text{FoV}_y}{2}\right) $$
    Where $\text{FoV}_x$ and $\text{FoV}_y$ are the horizontal and vertical field of view angles in radians.

2.  **Principal Point:**
    $$ cx = \text{intrinsic}[0, 2] $$
    $$ cy = \text{intrinsic}[1, 2] $$
    Where `intrinsic` is the camera intrinsic matrix.

3.  **Matrices:**
    *   `viewmatrix`: Extracted from `viewpoint_camera.world_view_transform`.
    *   `projmatrix`: Extracted from the camera projection matrix.

### Pseudocode
The following Python-like pseudocode demonstrates the creation of `GaussianRasterizationSettings` and the setup of auxiliary tensors required for the rasterization pipeline [S4, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def create_rasterization_settings(viewpoint_camera, image_height, image_width, bg_color, scale_modifier=1.0, sh_degree=2, return_depth=False):
    """
    Creates GaussianRasterizationSettings configuration.
    
    Args:
        viewpoint_camera: Object containing FoVx, FoVy, world_view_transform, projection_matrix, etc.
        image_height: int
        image_width: int
        bg_color: torch.Tensor
        scale_modifier: float, default 1.0
        sh_degree: int
        return_depth: bool
        
    Returns:
        GaussianRasterizationSettings instance
    """
    # Compute FOV tangents
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    
    # Extract principal point from intrinsic matrix if available, otherwise assume center
    # Note: Source [S1, unofficial] specifies extraction from intrinsic[0,2] and intrinsic[1,2]
    # If intrinsic is not directly available, cx/cy might be derived from image dimensions
    if hasattr(viewpoint_camera, 'intrinsic'):
        cx = viewpoint_camera.intrinsic[0, 2]
        cy = viewpoint_camera.intrinsic[1, 2]
    else:
        cx = image_width / 2.0
        cy = image_height / 2.0
        
    # Extract matrices
    viewmatrix = viewpoint_camera.world_view_transform
    projmatrix = viewpoint_camera.projection_matrix
    campos = viewpoint_camera.camera_center # Assuming this attribute exists
    
    # Create settings object
    settings = GaussianRasterizationSettings(
        image_height=int(image_height),
        image_width=int(image_width),
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        cx=float(cx),
        cy=float(cy),
        bg=bg_color,
        scale_modifier=scale_modifier,
        viewmatrix=viewmatrix,
        projmatrix=projmatrix,
        sh_degree=sh_degree,
        campos=campos,
        prefiltered=False, # Default value, check source for specific usage
        backward_geometry=True, # Enable geometry gradients
        computer_pseudo_normal=False, # Note: typo in source [S1, unofficial]
        debug=False
    )
    
    return settings

def setup_screenspace_points(gaussian_xyz):
    """
    Sets up the screenspace_points tensor for gradient accumulation.
    """
    # Initialize to zero with same dtype and device as xyz
    screenspace_points = torch.zeros_like(gaussian_xyz, dtype=gaussian_xyz.dtype, device=gaussian_xyz.device)
    
    # Attempt to retain gradients for older PyTorch versions
    try:
        screenspace_points.retain_grad()
    except AttributeError:
        pass # Fallback for older versions
        
    return screenspace_points
```

### Edge Cases and Pitfalls
*   **Precision:** Field of view tangents should be computed using `math.tan` to avoid precision issues with small angles.
*   **Device Consistency:** All tensors (`bg`, `viewmatrix`, `projmatrix`, `campos`) must reside on the same device (typically CUDA) as the Gaussian attributes to prevent kernel launch errors [Parent].
*   **Missing Tensors:** Optional tensors (like `campos` if not explicitly stored) should be handled gracefully. Missing optional tensors in the broader pipeline are replaced with empty tensors to maintain batch consistency [Parent].
*   **Typo:** The field `computer_pseudo_normal` is a known typo in the source code [S1, unofficial]. Implementers should map this to `compute_pseudo_normal` if their rasterizer expects the corrected name.

## Subtopics

### 3D Half-Gaussian Rasterization
A variant where each 3D Gaussian is split into two halves by a plane through its center, allowing distinct opacity values ($\alpha_1, \alpha_2$) for each half to better represent sharp edges and planar surfaces [S2]. It is chosen when high-fidelity reconstruction of planar surfaces or sharp edges is required. The splitting plane normal $\mathbf{n} = [n_1, n_2, n_3]^T$ is stored using the three unused normal parameters in the original 3D-GS kernel implementation [S2]. The opacity for the first half ($\mathbf{n}^T(\mathbf{x}-\boldsymbol{\mu}) \ge 0$) is $\alpha_1$, and for the second half ($\mathbf{n}^T(\mathbf{x}-\boldsymbol{\mu}) < 0$) is $\alpha_2$ [S2]. This increases the parameter count by only one additional opacity term per Gaussian compared to standard 3D-GS [S2].

### Restricting Gaussian Kernel Support
A technique to reduce Gaussian footprints and superfluous ray-Gaussian intersections using a generalized kernel exponent and adaptive alpha clamping [S3]. It is chosen to improve rendering efficiency and reduce artifacts from large, sparse Gaussians. The generalized Gaussian kernel is defined as:
$$ \exp \left[ -\frac{1}{2n_{\text{expo}}} ((\mathbf{x} - \boldsymbol{\mu})^T \Sigma^{-1} (\mathbf{x} - \boldsymbol{\mu}))^{n_{\text{expo}}} \right] $$
where $n_{\text{expo}}$ controls the bulkiness [S3]. A pitfall is the denominator choice: 3DGRT uses $4.5 \cdot 2^{n_{\text{expo}}}$ while GRay uses $1 \cdot 2^{n_{\text{expo}}}$, with the latter making initialization less sensitive and resulting in smaller initial Gaussians [S3]. Dynamic clamping can cause visible artifacts for large Gaussians, though this is less apparent in differentiable rendering (DI) where Gaussians tend to stay small [S3].

### Gaussian Rasterization Configuration Setup
This refers to the procedural steps of initializing the rasterization settings within the render function, including the creation of auxiliary tensors like `screenspace_points` [S4, unofficial]. It is chosen when setting up a new training or rendering loop. Key steps include computing `tanfovx`/`tanfovy`, initializing `GaussianRasterizationSettings`, and creating a `screenspace_points` tensor with `requires_grad=True` to retain gradients for position updates [S4, unofficial]. The `return_depth` flag is used to control whether depth information is returned alongside the color render [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The creation of `GaussianRasterizationSettings` is $O(1)$ as it involves simple assignments and trigonometric computations. The actual rasterization time depends on the number of Gaussians $N$ and the image resolution $H \times W$, typically $O(N)$ for splatting and $O(H \times W)$ for compositing.
*   **Memory Complexity:** The settings object itself uses negligible memory. The `screenspace_points` tensor requires $O(N)$ memory for gradient storage [S4, unofficial].
*   **Strengths:** Centralizes camera and rendering parameters, simplifying the API for the CUDA kernel. Supports flexible background colors and scale modifications.
*   **Limitations:** The structure is rigid; adding new parameters requires modifying the `NamedTuple` definition and the CUDA kernel signature. The typo `computer_pseudo_normal` may cause confusion [S1, unofficial].
*   **When to Use:** Use when configuring the rendering pipeline for 3D Gaussian Splatting. Do not use if you need to dynamically change camera parameters per Gaussian (use per-Gaussian attributes instead).

## Background
The `GaussianRasterizationSettings` structure originates from the 3D Gaussian Splatting (3D-GS) framework introduced by Kerbl et al. [2023]. It was designed to efficiently pass camera parameters to the differentiable rasterizer. The specific implementation details, including the `NamedTuple` structure and field names, are derived from various open-source implementations and unofficial repositories [S1, S4, unofficial]. The concept of splitting Gaussians for better edge representation was introduced in 3D-HGS [S2]. The technique for restricting kernel support was proposed in 3DGRT and GRay [S3].

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] 3D-HGS: 3D Half-Gaussian Splatting*, https://doi.org/10.1109/cvpr52734.2025.01027, fetched 2026-10-30
[S3] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
[S4] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
