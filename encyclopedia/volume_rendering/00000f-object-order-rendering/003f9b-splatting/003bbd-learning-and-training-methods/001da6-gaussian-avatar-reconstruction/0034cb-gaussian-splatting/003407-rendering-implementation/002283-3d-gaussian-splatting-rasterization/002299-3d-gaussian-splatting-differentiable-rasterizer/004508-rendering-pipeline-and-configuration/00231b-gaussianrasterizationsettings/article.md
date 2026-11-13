# GaussianRasterizationSettings

Also called: 4D Gaussian Splatting Rasterization, GaussianRasterizationSettings configuration, GaussianRasterizer

## Overview
`GaussianRasterizationSettings` is a configuration structure (typically a `NamedTuple` or struct) that defines the camera parameters, background, and rendering flags required by the differentiable 3D Gaussian splatting rasterizer [S1, S2, S3, unofficial]. It serves as the bridge between the scene representation (Gaussians) and the CUDA kernel, ensuring that the rasterizer has all necessary projection data (view/projection matrices, field of view) and state (background color, SH degree) to perform forward and backward passes [S2, S3, unofficial]. It is distinct from the rasterization kernel itself; it sets up the state that the kernel consumes [Parent].

## How it works
The settings object is instantiated with camera intrinsics/extrinsics and rendering options. It is passed to the `GaussianRasterizer` (or `diff_gaussian_rasterization` backend) which validates the inputs against the scene data before launching the CUDA kernel [S2, unofficial].

### Inputs and Outputs
*   **Inputs**: Camera matrices (`viewmatrix`, `projmatrix`), intrinsics (`tanfovx`, `tanfovy`, `cx`, `cy`), background color (`bg`), and rendering flags (`sh_degree`, `rot_4d`, etc.).
*   **Outputs**: The settings object itself does not produce rendering outputs; it is consumed by the rasterizer to produce color, depth, flow, and gradients [S2, unofficial].

### Data Structures
The structure varies slightly between implementations (3D-GS vs 4D-GS vs FastGS), but shares core fields.

**Standard 3D-GS / FastGS Layout [S1, S3, unofficial]:**
*   `image_height`: int
*   `image_width`: int
*   `tanfovx`: float (Tangent of half horizontal FOV)
*   `tanfovy`: float (Tangent of half vertical FOV)
*   `cx`: float (Principal point x)
*   `cy`: float (Principal point y)
*   `bg`: torch.Tensor (Background color, shape `(3,)` or `(N, 3)`)
*   `scale_modifier`: float (Multiplier for Gaussian scales)
*   `viewmatrix`: torch.Tensor (4x4 View matrix)
*   `projmatrix`: torch.Tensor (4x4 Projection matrix)
*   `sh_degree`: int (Spherical Harmonics degree)
*   `campos`: torch.Tensor (Camera position)
*   `prefiltered`: bool
*   `backward_geometry`: bool (Enable geometry gradients)
*   `computer_pseudo_normal`: bool (Enable pseudo-normal computation) [S1, unofficial]
*   `debug`: bool

**4D-GS Layout [S2, unofficial]:**
*   `image_height`: int
*   `image_width`: int
*   `tanfovx`: float
*   `tanfovy`: float
*   `bg`: torch.Tensor
*   `scale_modifier`: float
*   `viewmatrix`: torch.Tensor
*   `projmatrix`: torch.Tensor
*   `sh_degree`: int
*   `sh_degree_t`: int (Time-varying SH degree)
*   `campos`: torch.Tensor
*   `timestamp`: float
*   `time_duration`: float
*   `rot_4d`: bool (Enable 4D rotation)
*   `gaussian_dim`: int
*   `force_sh_3d`: bool
*   `prefiltered`: bool
*   `debug`: bool

**FastGS Layout [S3, unofficial]:**
*   Includes `mult` (float), `get_flag` (bool), `metric_map` (torch.Tensor) in addition to standard fields.

### Numbered Steps for Configuration and Validation

1.  **Compute Intrinsics**: Calculate `tanfovx` and `tanfovy` from camera FOV.
    $$ \text{tanfovx} = \tan(\frac{\text{FoV}_x}{2}), \quad \text{tanfovy} = \tan(\frac{\text{FoV}_y}{2}) $$
    Extract principal points `cx`, `cy` from the intrinsic matrix $K$ (typically $K_{0,2}$ and $K_{1,2}$) [S1, unofficial].

2.  **Initialize Settings Object**: Create the `GaussianRasterizationSettings` instance with the computed intrinsics, view/projection matrices, and background color [S1, S2, unofficial].

3.  **Input Validation (Mutual Exclusivity)**: Before rasterization, the system validates that the scene data matches the settings:
    *   **Color**: Exactly one of `shs` (Spherical Harmonics) or `colors_precomp` (precomputed RGB) must be provided. If both are `None` or both are non-empty, raise an exception [S1, S2, S3, unofficial].
    *   **Geometry**: Exactly one of the parametric pair (`scales`, `rotations`) or `cov3D_precomp` (precomputed 3D covariance) must be provided. If both are `None` or both are non-empty, raise an exception [S1, S2, S3, unofficial].
    *   **4D Geometry**: If `rot_4d` is True and `cov3D_precomp` is None, then `rotations_r`, `scales_t`, and `ts` must all be non-None. Otherwise, raise an exception [S2, unofficial].

4.  **Empty Tensor Normalization**: Replace any `None` input tensors with empty `torch.Tensor([])` or shape $(N, 0)$ tensors to prevent `None` propagation into the CUDA kernel [S1, S2, S3, unofficial].
    *   Example: `if shs is None: shs = torch.Tensor([])` [S2, unofficial].

5.  **Argument Restructuring**: Pack the settings and scene data into a tuple for the CUDA kernel. The order for the forward pass is typically:
    `bg, means3D, colors_precomp, flow_2d, opacities, ts, scales, scales_t, rotations, rotations_r, scale_modifier, cov3Ds_precomp, prefilter_var, viewmatrix, projmatrix, tanfovx, tanfovy, image_height, image_width, sh, sh_degree, sh_degree_t, campos, timestamp, time_duration, rot_4d, gaussian_dim, force_sh_3d, prefiltered, debug` [S2, unofficial].

6.  **Kernel Execution**: Call `_C.rasterize_gaussians(*args)` [S2, unofficial]. If `debug` is True, copy args to CPU for error snapshotting [S2, unofficial].

7.  **Backward Pass**: The backward pass receives gradients from the loss and the saved buffers from the forward pass. It restructures arguments similarly and calls `_C.rasterize_gaussians_backward(*args)` [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def create_gaussian_rasterization_settings(camera, background_color, sh_degree=0, rot_4d=False):
    """
    Creates the GaussianRasterizationSettings object from a camera and background.
    """
    # 1. Compute Intrinsics
    # Assuming camera has FoVx, FoVy, and intrinsic matrix K
    tanfovx = math.tan(camera.FoVx * 0.5)
    tanfovy = math.tan(camera.FoVy * 0.5)
    
    # Extract principal point from intrinsic matrix K (3x3)
    cx = camera.intrinsic[0, 2]
    cy = camera.intrinsic[1, 2]
    
    # 2. Initialize Settings
    # Using a NamedTuple-like structure for clarity
    settings = {
        "image_height": camera.image_height,
        "image_width": camera.image_width,
        "tanfovx": tanfovx,
        "tanfovy": tanfovy,
        "cx": cx,
        "cy": cy,
        "bg": torch.tensor(background_color, dtype=torch.float32, device="cuda"),
        "scale_modifier": 1.0,
        "viewmatrix": camera.viewmatrix,
        "projmatrix": camera.projmatrix,
        "sh_degree": sh_degree,
        "campos": camera.position,
        "prefiltered": False,
        "backward_geometry": True,
        "computer_pseudo_normal": False,
        "debug": False,
        # 4D-GS specific fields (if applicable)
        "sh_degree_t": 0,
        "timestamp": 0.0,
        "time_duration": 1.0,
        "rot_4d": rot_4d,
        "gaussian_dim": 3,
        "force_sh_3d": False
    }
    return settings

def validate_and_prepare_inputs(settings, gaussians):
    """
    Validates inputs and prepares them for the rasterizer kernel.
    """
    shs = gaussians.shs
    colors_precomp = gaussians.colors_precomp
    scales = gaussians.scales
    rotations = gaussians.rotations
    cov3D_precomp = gaussians.cov3D_precomp
    
    # 3. Input Validation: Color
    if (shs is None and colors_precomp is None) or (shs is not None and colors_precomp is not None):
        raise Exception("Please provide exactly one of either SHs or precomputed colors!")
        
    # 3. Input Validation: Geometry
    if (scales is None or rotations is None) and cov3D_precomp is None:
        if not (scales is not None and rotations is not None):
             raise Exception("Please provide exactly one of either scale/rotation pair or precomputed 3D covariance!")
    if (scales is not None or rotations is not None) and cov3D_precomp is not None:
        raise Exception("Please provide exactly one of either scale/rotation pair or precomputed 3D covariance!")
        
    # 3. Input Validation: 4D Geometry
    if settings["rot_4d"] and cov3D_precomp is None:
        rotations_r = gaussians.rotations_r
        scales_t = gaussians.scales_t
        ts = gaussians.ts
        if rotations_r is None or scales_t is None or ts is None:
            raise Exception("Please provide exactly rotations_r and scales_t and ts if rot_4d and cov3D_precomp is None!")

    # 4. Empty Tensor Normalization
    if shs is None: shs = torch.Tensor([])
    if colors_precomp is None: colors_precomp = torch.Tensor([])
    if scales is None: scales = torch.Tensor([])
    if rotations is None: rotations = torch.Tensor([])
    if cov3D_precomp is None: cov3D_precomp = torch.Tensor([])
    # ... normalize other optional tensors like flow_2d, ts, etc. ...

    # 5. Argument Restructuring
    args = (
        settings["bg"],
        gaussians.means3D,
        colors_precomp,
        gaussians.flow_2d if hasattr(gaussians, 'flow_2d') else torch.Tensor([]),
        gaussians.opacities,
        ts if 'ts' in gaussians.__dict__ else torch.Tensor([]),
        scales,
        scales_t if 'scales_t' in gaussians.__dict__ else torch.Tensor([]),
        rotations,
        rotations_r if 'rotations_r' in gaussians.__dict__ else torch.Tensor([]),
        settings["scale_modifier"],
        cov3D_precomp,
        gaussians.prefilter_var if hasattr(gaussians, 'prefilter_var') else False,
        settings["viewmatrix"],
        settings["projmatrix"],
        settings["tanfovx"],
        settings["tanfovy"],
        settings["image_height"],
        settings["image_width"],
        shs,
        settings["sh_degree"],
        settings["sh_degree_t"],
        settings["campos"],
        settings["timestamp"],
        settings["time_duration"],
        settings["rot_4d"],
        settings["gaussian_dim"],
        settings["force_sh_3d"],
        settings["prefiltered"],
        settings["debug"]
    )
    return args
```

### Parameters with Typical Values
*   `tanfovx`, `tanfovy`: Derived from camera FOV. Typical values depend on the lens (e.g., 0.5–1.5 for wide angles).
*   `scale_modifier`: Default 1.0 [S1, unofficial].
*   `sh_degree`: Typically 0–3 for 3D-GS [S1, unofficial].
*   `rot_4d`: False for static scenes, True for dynamic 4D-GS [S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Floating point precision in `tanfovx`/`tanfovy` affects projection accuracy.
*   **Boundaries**: If `cx` or `cy` are outside the image bounds, the rasterizer may produce artifacts or empty images.
*   **Degenerate Input**: Providing both `shs` and `colors_precomp` or neither raises an exception [S2, unofficial].
*   **Device Consistency**: All tensors must be on the same device (CUDA) [Parent].

## Subtopics

### [Gaussian Rasterization Configuration Setup](10752-gaussian-rasterization-configuration-setup.md)
This procedure initializes the `GaussianRasterizationSettings` object required by the differentiable 3D Gaussian splatting rasterizer. It prepares the camera projection parameters and gradient buffers necessary for the forward and backward passes of the rendering pipeline. It is distinct from the rasterization kernel itself; it sets up the state that the kernel consumes [Parent].

### 3D Half-Gaussian Rasterization
A rasterization technique that represents a 3D Gaussian kernel as two halves split by a plane through its center, allowing distinct opacity values for each half to better represent sharp edges and planar surfaces [S4].
*   **Parameters**: Splitting plane normal $n = [n_1, n_2, n_3]^T$ (learned unit vector, stored in unused normal parameters of 3D-GS kernel) [S4]. Opacity $\alpha_1$ for $n^T(x-\mu) \ge 0$ and $\alpha_2$ for $n^T(x-\mu) < 0$ [S4].
*   **When to choose**: When sharp edges and planar surfaces are prominent in the scene, and standard 3D-GS fails to capture them accurately [S4].

### Restricting Gaussian Kernel Support
Technique to reduce Gaussian footprints and superfluous ray-Gaussian intersections using a generalized kernel exponent and adaptive alpha clamping [S5].
*   **Formula**: Generalized Gaussian kernel: $\exp \left[ -\frac{1}{2n_{\text{expo}}} ((x - \mu)^T \Sigma^{-1} (x - \mu))^{n_{\text{expo}}} \right]$, where $n_{\text{expo}}$ is the exponent controlling bulkiness [S5].
*   **Pitfalls**: 3DGRT uses $4.5 n_{\text{expo}}$ instead of $1 2^{n_{\text{expo}}}$ in the denominator; GRay uses $1 2^{n_{\text{expo}}}$ to make initialization less sensitive and result in smaller initial Gaussians [S5]. Dynamic clamping causes visible artifacts for large Gaussians, but these are less apparent when using DI since most Gaussians stay small [S5].
*   **When to choose**: When rendering scenes with high Gaussian counts and needing to reduce ray-Gaussian intersection overhead [S5].

## Operating characteristics
*   **Time Complexity**: The rasterization step is $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels, but optimized via visibility culling and binning to be much faster in practice [S2, unofficial].
*   **Memory Complexity**: $O(N)$ for storing Gaussian parameters and $O(P)$ for the output image [S2, unofficial].
*   **Performance**: Real-time rendering is achievable on modern GPUs (e.g., RTX 4090) for scenes with millions of Gaussians at 1080p resolution [S5].
*   **Strengths**: Differentiable, fast, supports dynamic scenes (4D-GS) [S2, unofficial].
*   **Limitations**: Memory intensive for very large scenes; artifacts can occur with large Gaussians if not clamped [S5].

## Background
Originated from the 3D Gaussian Splatting paper [Parent]. Extended to 4D Gaussian Splatting by Yang et al. (ICLR 2024) to model dynamic scenes [S2, unofficial]. Variants like 3D Half-Gaussian Splatting [S4] and GRay [S5] have been proposed to improve quality and performance.

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S3] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
[S4] 3D-HGS: 3D Half-Gaussian Splatting*, https://doi.org/10.1109/cvpr52734.2025.01027, fetched 2026-10-30
[S5] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
