# GaussianRasterizationSettings

Also called: 4D Gaussian Splatting Rasterization, GaussianRasterizationSettings configuration, GaussianRasterizer

## Overview
`GaussianRasterizationSettings` is a configuration structure that bundles camera intrinsics, extrinsics, and rasterization flags required by the differentiable 3D Gaussian splatting rasterizer. It serves as the primary input interface for the rendering pipeline, defining how 3D Gaussians are projected and composited onto the 2D image plane. It is distinct from the rasterization kernel itself; it sets up the state that the kernel consumes [S2, unofficial]. It builds on [Rendering Pipeline and Configuration](11406-rendering-pipeline-and-configuration.md) by providing the specific parameter bundle that the parent's shared principles enforce (mutual exclusivity of geometry/color representations and device consistency).

## How it works
The settings object is typically implemented as a `NamedTuple` (PyTorch) or `struct` (C++/CUDA) that is passed to the CUDA rasterization kernels (`_C.rasterize_gaussians` and `_C.rasterize_gaussians_backward`). The rasterizer uses these settings to determine projection matrices, background color, spherical harmonic degree, and dynamic scene parameters (for 4D variants).

### Inputs and Outputs
*   **Inputs:** Camera matrices (`viewmatrix`, `projmatrix`), intrinsics (`tanfovx`, `tanfovy`, `cx`, `cy`), image dimensions (`image_height`, `image_width`), background color tensor (`bg`), and flags controlling rendering behavior (`sh_degree`, `rot_4d`, `debug`, etc.).
*   **Outputs:** The settings object itself does not produce rendering outputs; it is consumed by the rasterizer to produce color, depth, flow, and gradient tensors.

### Data Structures
The exact layout varies slightly between implementations (3D GS vs. 4D GS vs. variants), but the core fields are consistent.

**Standard 3D Gaussian Splatting Layout [S1, unofficial]:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
GaussianRasterizationSettings = NamedTuple(
    image_height: int,
    image_width: int,
    tanfovx: float,
    tanfovy: float,
    cx: float,
    cy: float,
    bg: torch.Tensor,       # Shape (3,) or (4,), device CUDA
    scale_modifier: float,
    viewmatrix: torch.Tensor, # 4x4
    projmatrix: torch.Tensor, # 4x4
    sh_degree: int,
    campos: torch.Tensor,     # Camera position, Shape (3,)
    prefiltered: bool,
    backward_geometry: bool,
    computer_pseudo_normal: bool,
    debug: bool
)
```

**4D Gaussian Splatting Layout [S2, unofficial]:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
GaussianRasterizationSettings = NamedTuple(
    image_height: int,
    image_width: int,
    tanfovx: float,
    tanfovy: float,
    bg: torch.Tensor,
    scale_modifier: float,
    viewmatrix: torch.Tensor,
    projmatrix: torch.Tensor,
    sh_degree: int,
    sh_degree_t: int,         # Temporal SH degree
    campos: torch.Tensor,
    timestamp: float,         # Current time step
    time_duration: float,     # Total time duration
    rot_4d: bool,             # Enable 4D rotation
    gaussian_dim: int,
    force_sh_3d: bool,
    prefiltered: bool,
    debug: bool
)
```

**FastGS/Laplacian GS Layout [S3, unofficial]:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
GaussianRasterizationSettings = NamedTuple(
    image_height: int,
    image_width: int,
    tanfovx: float,
    tanfovy: float,
    bg: torch.Tensor,
    scale_modifier: float,
    viewmatrix: torch.Tensor,
    projmatrix: torch.Tensor,
    sh_degree: int,
    campos: torch.Tensor,
    mult: float,
    prefiltered: bool,
    debug: bool,
    get_flag: bool,
    metric_map: torch.Tensor
)
```

### Parameters and Typical Values
*   **`tanfovx`, `tanfovy`**: Tangent of half horizontal/vertical field of view. Computed as `math.tan(camera.FoVx * 0.5)` and `math.tan(camera.FoVy * 0.5)` [S1, unofficial].
*   **`cx`, `cy`**: Principal point coordinates in pixels. Extracted from `intrinsic[0, 2]` and `intrinsic[1, 2]` [S1, unofficial].
*   **`scale_modifier`**: Multiplicative scalar applied to Gaussian scales during rasterization. Default value is `1.0` [S1, unofficial].
*   **`bg`**: Background color tensor. Typically shape `(3,)` for RGB or `(4,)` for RGBA, residing on CUDA. Default is often `[1, 1, 1]` (white) or `[0, 0, 0]` (black) [S2, unofficial].
*   **`sh_degree`**: Maximum spherical harmonic degree. Typically `3` for high-quality rendering [S2, unofficial].
*   **`rot_4d`**: Boolean flag to enable 4D rotation (time-varying). If `True`, the rasterizer expects additional inputs (`rotations_r`, `scales_t`, `ts`) [S2, unofficial].
*   **`debug`**: Boolean flag. If `True`, the rasterizer copies arguments to CPU for error dumping (`snapshot_fw.dump` or `snapshot_bw.dump`) upon failure [S2, unofficial].

### Forward Pass Execution
The rasterizer performs the following steps when invoked with `GaussianRasterizationSettings`:

1.  **Input Validation**:
    *   Check color representation: Exactly one of `shs` (Spherical Harmonics) or `colors_precomp` (precomputed colors) must be provided. If neither or both are provided, raise Exception [S2, unofficial].
    *   Check geometry representation: Exactly one of (`scales` and `rotations`) or `cov3D_precomp` (precomputed 3D covariance) must be provided. If neither or both are provided, raise Exception [S2, unofficial].
    *   Check 4D requirements: If `rot_4d` is `True` and `cov3D_precomp` is `None`, then `rotations_r`, `scales_t`, and `ts` must all be non-None. Otherwise, raise Exception [S2, unofficial].
2.  **Initialization**: Replace any `None` optional tensors with empty tensors (e.g., `torch.Tensor([])`) to maintain batch consistency [S2, unofficial].
3.  **Argument Restructuring**: Construct a tuple of arguments in the specific order expected by the CUDA kernel `_C.rasterize_gaussians`. The order typically includes: `bg`, `means3D`, `colors_precomp`, `flow_2d`, `opacities`, `ts`, `scales`, `scales_t`, `rotations`, `rotations_r`, `scale_modifier`, `cov3Ds_precomp`, `prefilter_var`, `viewmatrix`, `projmatrix`, `tanfovx`, `tanfovy`, `image_height`, `image_width`, `sh`, `sh_degree`, `sh_degree_t`, `campos`, `timestamp`, `time_duration`, `rot_4d`, `gaussian_dim`, `force_sh_3d`, `prefiltered`, `debug` [S2, unofficial].
4.  **Kernel Execution**:
    *   If `debug` is `True`, copy args to CPU (`cpu_args`) using `cpu_deep_copy_tuple`.
    *   Call `_C.rasterize_gaussians(*args)`.
    *   If an exception occurs, save `cpu_args` to `'snapshot_fw.dump'` and re-raise [S2, unofficial].
5.  **Context Saving**: Save `raster_settings`, `num_rendered`, `prefilter_var`, and all input tensors (`colors_precomp`, `means3D`, `out_means3D`, `scales`, `rotations`, `cov3Ds_precomp`, `radii`, `sh`, `flow_2d`, `opacities`, `ts`, `scales_t`, `rotations_r`, `geomBuffer`, `binningBuffer`, `imgBuffer`) for the backward pass [S2, unofficial].
6.  **Return**: Return `(color, radii, depth, 1-T, flow, covs_com)` where `T` is the accumulated transmittance [S2, unofficial].

### Backward Pass Execution
1.  **Argument Restructuring**: Construct args tuple in order: `bg`, `means3D`, `out_means3D`, `radii`, `colors_precomp`, `flow_2d`, `opacities`, `ts`, `scales`, `scales_t`, `rotations`, `rotations_r`, `scale_modifier`, `cov3Ds_precomp`, `prefilter_var`, `viewmatrix`, `projmatrix`, `tanfovx`, `tanfovy`, `grad_out_color`, `grad_depth`, `grad_alpha`, `grad_flow`, `sh`, `sh_degree`, `sh_degree_t`, `campos`, `timestamp`, `time_duration`, `rot_4d`, `gaussian_dim`, `force_sh_3d`, `geomBuffer`, `num_rendered`, `binningBuffer`, `imgBuffer`, `debug` [S2, unofficial].
2.  **Kernel Execution**:
    *   If `debug` is `True`, copy args to CPU.
    *   Call `_C.rasterize_gaussians_backward(*args)`.
    *   If an exception occurs, save `cpu_args` to `'snapshot_bw.dump'` and re-raise [S2, unofficial].
3.  **Return**: Return gradients tuple: `(grad_means3D, grad_means2D, grad_sh, grad_colors_precomp, grad_flows, grad_opacities, grad_ts, grad_scales, grad_scales_t, grad_rotations, grad_rotations_r, grad_cov3Ds_precomp, None, None)` [S2, unofficial].

### Visibility Marking
A separate utility function `mark_visible` is often used to pre-calculate which Gaussians are within the camera frustum.
1.  Call `_C.mark_visible(positions, viewmatrix, projmatrix)` with `torch.no_grad()`.
2.  Returns a boolean tensor indicating visible Gaussians [S2, unofficial].

### Edge Cases and Pitfalls
*   **Mutual Exclusivity**: The rasterizer strictly enforces that only one geometry representation and one color representation are provided. Providing both or neither raises an exception [S2, unofficial].
*   **4D Requirements**: When `rot_4d` is enabled, the absence of `rotations_r`, `scales_t`, or `ts` when `cov3D_precomp` is not provided causes a runtime error [S2, unofficial].
*   **Debug Mode**: Enabling `debug` significantly increases memory usage and latency due to CPU-GPU transfers and file I/O on error. It should only be used during development [S2, unofficial].
*   **Precision**: All tensors must reside on the same device (typically CUDA) to prevent kernel launch errors [Parent].

## Subtopics

### 3D Half-Gaussian Rasterization
A rasterization technique that represents a 3D Gaussian kernel as two halves split by a plane through its center, allowing distinct opacity values for each half to better represent sharp edges and planar surfaces [S4]. It sets itself apart by introducing a learned splitting plane normal `n` and two opacity parameters `α1` and `α2` [S4]. Choose this when rendering scenes with sharp planar features or edges where standard Gaussians blur details. The splitting plane normal `n = [n1, n2, n3]^T` is stored using the three unused normal parameters in the original 3D-GS kernel implementation [S4]. Opacity `α1` applies where `n^T(x-μ) ≥ 0` and `α2` where `n^T(x-μ) < 0` [S4].

### Restricting Gaussian Kernel Support
Techniques to reduce Gaussian footprints and superfluous ray-Gaussian intersections using a generalized kernel exponent and adaptive alpha clamping [S5]. It sets itself apart by modifying the kernel shape and support threshold rather than the rasterization pipeline itself. Choose this when dealing with large Gaussians that cause excessive overlap or artifacts. The generalized Gaussian kernel is defined as $\exp \left[ -\frac{1}{2n_{\text{expo}}} ((x - \mu)^T \Sigma^{-1} (x - \mu))^{n_{\text{expo}}} \right]$, where $n_{\text{expo}}$ controls bulkiness [S5]. A pitfall is that 3DGRT uses $4.5 n_{\text{expo}}$ in the denominator, while GRay uses $1 2^{n_{\text{expo}}}$ to make initialization less sensitive and result in smaller initial Gaussians [S5]. Dynamic clamping can cause visible artifacts for large Gaussians, but these are less apparent when using DI since most Gaussians stay small [S5].

## Operating characteristics
*   **Time Complexity**: The rasterization step is $O(N)$ with respect to the number of Gaussians $N$, but the constant factor is high due to sorting, projection, and compositing. The visibility marking step is also $O(N)$.
*   **Memory Complexity**: $O(N)$ for storing Gaussian parameters and intermediate buffers (`geomBuffer`, `binningBuffer`, `imgBuffer`). The backward pass requires storing all input tensors and intermediate results, doubling memory usage during training [S2, unofficial].
*   **Measured Performance**: Real-time performance (30+ FPS) is achievable on modern GPUs (e.g., RTX 4090) for scenes with up to a few million Gaussians at 1080p resolution [S5]. Performance scales linearly with Gaussian count but is sensitive to the number of visible Gaussians per pixel.
*   **Strengths**: Differentiable, real-time rendering, high visual quality with few primitives.
*   **Limitations**: High memory usage during training, sensitive to hyperparameters (e.g., `scale_modifier`, `sh_degree`), and requires careful handling of 4D dynamics [S2, unofficial].
*   **When to Use**: Real-time novel view synthesis, dynamic scene rendering, and differentiable rendering pipelines.
*   **When Not to Use**: Offline rendering where ray tracing is preferred for accuracy, or scenes with extremely high Gaussian counts that exceed GPU memory.

## Background
`GaussianRasterizationSettings` originated with the introduction of 3D Gaussian Splatting (3D-GS) by Kerbl et al. in 2023 [S1, unofficial]. It was extended to 4D Gaussian Splatting by Yang et al. in 2024 to handle dynamic scenes [S2, unofficial]. The structure has been adapted in various implementations (FastGS, Laplacian-GS, 4D-GS) to support additional features like temporal spherical harmonics, 4D rotations, and sparse optimization [S2, unofficial][S3, unofficial]. It is widely used in real-time photorealistic dynamic scene representation and rendering [S2, unofficial].

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S3] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
[S4] 3D-HGS: 3D Half-Gaussian Splatting*, https://doi.org/10.1109/cvpr52734.2025.01027, fetched 2026-10-30
[S5] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
