# GaussianRasterizationSettings

Also called: 4D Gaussian Splatting Rasterization, GaussianRasterizationSettings configuration, GaussianRasterizer

## Overview
`GaussianRasterizationSettings` is a configuration structure (typically a `NamedTuple` or struct) that encapsulates the camera parameters, scene constraints, and rendering flags required to invoke the differentiable 3D Gaussian rasterizer [S1, S2, S3, unofficial]. It solves the problem of translating high-level camera definitions (Field of View, view/projection matrices) and scene properties into the specific numerical formats expected by the underlying CUDA kernel [S1, unofficial]. It differs from the rasterizer itself by handling the setup of state and memory layout rather than the pixel-wise projection and blending logic [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
The rasterizer receives the settings object along with per-Gaussian attributes:
- `means3D`: $(N, 3)$ tensor of 3D positions.
- `opacities`: $(N,)$ tensor of scalar opacities.
- `scales` / `rotations`: $(N, 3)$ and $(N, 4)$ tensors for scale and quaternion rotation, OR `cov3D_precomp`: $(N, 6)$ tensor of pre-computed 3D covariance matrix elements [S2, unofficial].
- `shs` / `colors_precomp`: Spherical Harmonics coefficients or pre-computed RGB colors [S2, unofficial].
- Camera parameters: `viewmatrix`, `projmatrix`, `tanfovx`, `tanfovy`, `campos` [S2, unofficial].
- Optional dynamic parameters: `ts` (time), `scales_t`, `rotations_r` for 4D Gaussians [S2, unofficial].

**Outputs:**
- `color`: Rendered image tensor.
- `radii`: Visibility mask for Gaussians.
- `depth`: Per-pixel depth.
- `flow`: Optional 2D flow field [S2, unofficial].
- Gradients for all input attributes [S2, unofficial].

### Data Structures
The settings object varies slightly between implementations (e.g., 3D-GS vs. 4D-GS), but generally contains:

**Standard Fields (3D-GS):**
- `image_height` (int): Height of the output image in pixels.
- `image_width` (int): Width of the output image in pixels.
- `tanfovx` (float): $\tan(\text{FoVx} / 2)$.
- `tanfovy` (float): $\tan(\text{FoVy} / 2)$.
- `bg` (torch.Tensor): Background color tensor, shape $(3,)$.
- `scale_modifier` (float): Multiplicative scalar applied to Gaussian scales. Default 1.0 [S1, unofficial].
- `viewmatrix` (torch.Tensor): $4 \times 4$ view transformation matrix.
- `projmatrix` (torch.Tensor): $4 \times 4$ projection matrix.
- `sh_degree` (int): Degree of Spherical Harmonics.
- `campos` (torch.Tensor): Camera position in world space, derived from view matrix $V = [R | t; 0, 1]$ as $\text{campos} = -R^T t$ [S2, unofficial].
- `prefiltered` (bool): Flag indicating if SH coefficients are pre-filtered.
- `debug` (bool): If true, copies arguments to CPU for debugging snapshots on error [S2, unofficial].

**Extended Fields (4D-GS):**
- `sh_degree_t` (int): Time-dependent SH degree [S2, unofficial].
- `timestamp` (float): Current time step [S2, unofficial].
- `time_duration` (float): Total duration of the dynamic scene [S2, unofficial].
- `rot_4d` (bool): Flag enabling 4D rotation modeling [S2, unofficial].
- `gaussian_dim` (int): Dimensionality of the Gaussian representation [S2, unofficial].
- `force_sh_3d` (bool): Force 3D SH evaluation even in 4D mode [S2, unofficial].

*Note: Some variants include `cx`, `cy` (principal point), `backward_geometry`, `computer_pseudo_normal`, `mult`, `get_flag`, `metric_map` [S1, S3, unofficial].*

### Numbered Steps

1.  **Input Validation**:
    -   Check color representation: Exactly one of `shs` or `colors_precomp` must be provided. If neither or both are provided, raise `Exception 'Please provide excatly one of either SHs or precomputed colors!'` [S2, unofficial].
    -   Check geometry representation: Exactly one of the scale/rotation pair (`scales`, `rotations`) or `cov3D_precomp` must be provided. If neither or both are provided, raise `Exception 'Please provide exactly one of either scale/rotation pair or precomputed 3D covariance!'` [S2, unofficial].
    -   Check 4D requirements: If `rot_4d` is True and `cov3D_precomp` is None, then `rotations_r`, `scales_t`, and `ts` must all be non-None. Otherwise, raise `Exception 'Please provide exactly rotations_r and scales_t and ts if rot_4d and cov3D_precomp is None!'` [S2, unofficial].

2.  **Initialization (None Handling)**:
    -   Convert any `None` input tensors to empty `torch.Tensor` objects to prevent CUDA kernel crashes [S2, unofficial]. Specifically:
        ```python
        if shs is None: shs = torch.Tensor([])
        if colors_precomp is None: colors_precomp = torch.Tensor([])
        if flow_2d is None: flow_2d = torch.Tensor([])
        if ts is None: ts = torch.Tensor([])
        if scales is None: scales = torch.Tensor([])
        if scales_t is None: scales_t = torch.Tensor([])
        if rotations is None: rotations = torch.Tensor([])
        if rotations_r is None: rotations_r = torch.Tensor([])
        if cov3D_precomp is None: cov3D_precomp = torch.Tensor([])
        ```
    [S2, unofficial].

3.  **Argument Restructuring (Forward)**:
    -   Construct the `args` tuple in the exact order expected by the C++/CUDA backend `_C.rasterize_gaussians`:
        ```python
        args = (
            raster_settings.bg, means3D, colors_precomp, flow_2d, opacities, ts,
            scales, scales_t, rotations, rotations_r, raster_settings.scale_modifier,
            cov3Ds_precomp, prefilter_var, raster_settings.viewmatrix, raster_settings.projmatrix,
            raster_settings.tanfovx, raster_settings.tanfovy, raster_settings.image_height,
            raster_settings.image_width, sh, raster_settings.sh_degree, raster_settings.sh_degree_t,
            raster_settings.campos, raster_settings.timestamp, raster_settings.time_duration,
            raster_settings.rot_4d, raster_settings.gaussian_dim, raster_settings.force_sh_3d,
            raster_settings.prefiltered, raster_settings.debug
        )
        ```
    [S2, unofficial].

4.  **Forward Execution**:
    -   If `raster_settings.debug` is True, perform a deep copy of `args` to CPU (`cpu_args`) [S2, unofficial].
    -   Call `_C.rasterize_gaussians(*args)`.
    -   If an exception occurs and debug is True, save `cpu_args` to `'snapshot_fw.dump'` and re-raise [S2, unofficial].
    -   Return `num_rendered`, `color`, `flow`, `depth`, `T`, `radii`, `geomBuffer`, `binningBuffer`, `imgBuffer`, `covs_com`, `out_means3D` [S2, unofficial].

5.  **Context Saving (Backward)**:
    -   Save `raster_settings`, `num_rendered`, `prefilter_var`.
    -   Save tensors for backward pass: `colors_precomp`, `means3D`, `out_means3D`, `scales`, `rotations`, `cov3Ds_precomp`, `radii`, `sh`, `flow_2d`, `opacities`, `ts`, `scales_t`, `rotations_r`, `geomBuffer`, `binningBuffer`, `imgBuffer` [S2, unofficial].

6.  **Backward Execution**:
    -   Restructure backward args:
        ```python
        bw_args = (
            raster_settings.bg, means3D, out_means3D, radii, colors_precomp, flow_2d, opacities, ts,
            scales, scales_t, rotations, rotations_r, raster_settings.scale_modifier, cov3Ds_precomp,
            prefilter_var, raster_settings.viewmatrix, raster_settings.projmatrix, raster_settings.tanfovx,
            raster_settings.tanfovy, grad_out_color, grad_depth, grad_alpha, grad_flow, sh,
            raster_settings.sh_degree, raster_settings.sh_degree_t, raster_settings.campos,
            raster_settings.timestamp, raster_settings.time_duration, raster_settings.rot_4d,
            raster_settings.gaussian_dim, raster_settings.force_sh_3d, geomBuffer, num_rendered,
            binningBuffer, imgBuffer, raster_settings.debug
        )
        ```
    [S2, unofficial].
    -   Call `_C.rasterize_gaussians_backward(*bw_args)`.
    -   Return gradients: `grad_means3D`, `grad_means2D`, `grad_sh`, `grad_colors_precomp`, `grad_flows`, `grad_opacities`, `grad_ts`, `grad_scales`, `grad_scales_t`, `grad_rotations`, `grad_rotations_r`, `grad_cov3Ds_precomp`, `None`, `None` [S2, unofficial].

### Parameters with Typical Values
-   `tanfovx`, `tanfovy`: Computed as `math.tan(camera.FoVx * 0.5)` and `math.tan(camera.FoVy * 0.5)` [S1, unofficial].
-   `scale_modifier`: Default 1.0 [S1, unofficial].
-   `bg`: Typically `[1, 1, 1]` for white background or `[0, 0, 0]` for black [S2, unofficial].

### Edge Cases and Pitfalls
-   **None Tensors**: Passing `None` directly to the CUDA kernel causes crashes. Must be converted to `torch.Tensor([])` [S2, unofficial].
-   **Device Consistency**: All input tensors must reside on the same device (typically CUDA) as the rasterizer's output. Mismatched devices cause undefined behavior [S1, Parent, unofficial].
-   **Debug Snapshots**: In debug mode, arguments are copied to CPU. This adds overhead but allows saving `snapshot_fw.dump` or `snapshot_bw.dump` on failure [S2, unofficial].
-   **Mutual Exclusivity**: Providing both SH and pre-computed colors, or both scale/rotation and pre-computed covariance, raises an exception [S2, unofficial].

## Subtopics

### [Gaussian Rasterization Configuration Setup](10752-gaussian-rasterization-configuration-setup.md)
This method initializes the `GaussianRasterizationSettings` object and prepares auxiliary state tensors required to invoke the differentiable Gaussian rasterizer. It solves the problem of translating high-level camera parameters (Field of View) and scene constraints into the specific numerical formats expected by the CUDA kernel. It differs from the rasterizer itself by handling the setup of state and memory layout rather than the pixel-wise projection and blending logic.

### [GSCache DiffRasterizer Interface](18374-gscache-diffrasterizer-interface.md)
The GSCache DiffRasterizer Interface is a C++ class-based abstraction for managing the differentiable rasterization of 3D Gaussians. It encapsulates camera state, frame buffer management, and the forward/backward passes required for gradient-based optimization of Gaussian splatting parameters. It distinguishes itself by providing a unified interface that handles not only rendering but also the integration of optimization steps (densification, pruning) and data access for the Gaussian model parameters.

### 3D Half-Gaussian Rasterization
A rasterization technique that represents a 3D Gaussian kernel as two halves split by a plane through its center, allowing distinct opacity values for each half to better represent sharp edges and planar surfaces [S4]. The splitting plane normal $\mathbf{n} = [n_1, n_2, n_3]^T$ is a learned unit vector, stored using the three unused normal parameters in the original 3D-GS kernel implementation [S4]. Opacity $\alpha_1$ applies where $\mathbf{n}^T(\mathbf{x}-\boldsymbol{\mu}) \ge 0$, and $\alpha_2$ applies where $\mathbf{n}^T(\mathbf{x}-\boldsymbol{\mu}) < 0$ [S4]. This increases memory by only one additional parameter per kernel compared to 3D-GS [S4].

### Restricting Gaussian Kernel Support
Technique to reduce Gaussian footprints and superfluous ray-Gaussian intersections using a generalized kernel exponent and adaptive alpha clamping [S5]. The generalized Gaussian kernel is:
$$ \exp \left[ -\frac{1}{2n_{\text{expo}}} ((x - \mu)^T \Sigma^{-1} (x - \mu))^{n_{\text{expo}}} \right] $$
where $n_{\text{expo}}$ controls bulkiness [S5]. 3DGRT uses $4.5 n_{\text{expo}}$ in the denominator, while GRay uses $1 2^{n_{\text{expo}}}$ to make initialization less sensitive and result in smaller initial Gaussians [S5]. Dynamic clamping can cause visible artifacts for large Gaussians, though these are less apparent when using DI since most Gaussians stay small [S5].

## Operating characteristics
-   **Time Complexity**: Linear in the number of Gaussians $N$ and pixels $P$, dominated by projection and sorting steps in the CUDA kernel.
-   **Memory Complexity**: $O(N + P)$ for storing Gaussian attributes and intermediate buffers (geomBuffer, binningBuffer, imgBuffer) [S2, unofficial].
-   **Performance**: Real-time rendering on modern GPUs (e.g., RTX 4090) for scenes with hundreds of thousands of Gaussians [S5].
-   **Strengths**: Differentiable, fast, supports dynamic scenes (4D-GS) [S2, unofficial].
-   **Limitations**: Sensitive to input tensor shapes and device placement; requires careful handling of `None` values [S2, unofficial].

## Background
Originating from the 3D Gaussian Splatting paper [S6, unofficial], the rasterization settings have been extended to support dynamic scenes in 4D Gaussian Splatting (ICLR 2024) [S2, unofficial]. The configuration structure ensures that the differentiable rasterizer receives consistent and valid parameters for forward and backward passes.

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S3] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
[S4] 3D-HGS: 3D Half-Gaussian Splatting*, https://doi.org/10.1109/cvpr52734.2025.01027, fetched 2026-10-30
[S5] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
[S6] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
