# ToF Gaussian Rasterization Forward Pass

Also called: ToF Gaussian Rasterization

## Overview
This method performs the forward pass of the C++/CUDA rasterizer for Time-of-Flight (ToF) Gaussian Splatting, extending standard 2D Gaussian Splatting to output phase and amplitude information alongside color and depth. It solves the problem of rendering raw ToF sensor data (phase, amplitude) by treating phasors as additional vector attributes of the 3D Gaussians, allowing for differentiable rendering of depth and intensity maps. It differs from its parent [2D Gaussian Splatting Rasterization](4815-2d-gaussian-splatting-rasterization.md) by explicitly handling complex-valued phasor accumulation and supporting phase/DC offset corrections, while differing from [Per-Primitive Texture Mapping for Reflective Gaussian Splatting](8629-per-primitive-texture-mapping-for-reflective-gaussian-splatt.md) by focusing on volumetric ToF signal synthesis rather than surface reflectance.

## How it works
**Inputs:**
- `raster_settings`: An object containing camera and rendering parameters.
- `gaussians`: A GaussianModel instance containing learnable and static attributes.
- `training_args`: Configuration flags (e.g., `optimize_phase_offset`).

**Data Structures:**
The method restructures arguments into a single tuple for the CUDA kernel. The layout is:
1. `raster_settings.bg`: Background tensor (typically 7 channels for ToF: RGB + Phase + Amplitude + Depth?).
2. `means3D`: 3D positions of Gaussians.
3. `colors_precomp`: Pre-computed color attributes.
4. `phasors_precomp`: Pre-computed phasor attributes (complex or real/imaginary pairs).
5. `opacities`: Scalar opacities.
6. `scales`: Scaling factors.
7. `rotations`: Quaternions.
8. `raster_settings.scale_modifier`: Scale modifier.
9. `cov3Ds_precomp`: Pre-computed 3D covariances.
10. `raster_settings.viewmatrix`: View matrix.
11. `raster_settings.projmatrix`: Projection matrix.
12. `raster_settings.tanfovx`, `raster_settings.tanfovy`: Tangent of field of view.
13. `raster_settings.image_height`, `raster_settings.image_width`: Image dimensions.
14. `sh`: Spherical Harmonics coefficients.
15. `sh_p`: Spherical Harmonics phase coefficients (if applicable).
16. `raster_settings.sh_degree`: SH degree.
17. `raster_settings.campos`: Camera position.
18. `raster_settings.prefiltered`: Prefiltering flag.
19. `raster_settings.debug`: Debug flag.
20. `raster_settings.near_n`, `raster_settings.far_n`: Near and far planes.
21. `raster_settings.depth_range`: Depth range.
22. `raster_settings.use_view_dependent_phase`: View-dependent phase flag.
23. `phase_offset`: Scalar phase offset.
24. `dc_offset`: Scalar DC offset.

**Parameters:**
- `phase_offset`: Scalar value for phase offset correction. If `raster_settings.optimize_phase_offset` is true, the raw tensor is used; otherwise, it is detached, converted to CPU numpy, and cast to a Python item (scalar) [S1, unofficial].
- `dc_offset`: Scalar value for DC offset correction. Logic mirrors `phase_offset` [S1, unofficial].

**Steps:**
1. **Argument Restructuring**: Construct the argument tuple as defined in the Data Structures section. Handle `phase_offset` and `dc_offset` based on optimization flags [S1, unofficial].
2. **Debug Handling**: If `raster_settings.debug` is true, deep-copy all arguments to CPU before invocation to preserve state for debugging. If an exception occurs, save the CPU copy to `snapshot_fw.dump` and print it [S1, unofficial].
3. **CUDA Invocation**: Call the C++/CUDA forward rasterization function `_C.rasterize_gaussians(*args)` [S1, unofficial].
4. **Backward Pass Preparation**: Save the following tensors for the backward pass: `raster_settings`, `num_rendered`, `colors_precomp`, `phasors_precomp`, `means3D`, `scales`, `rotations`, `cov3Ds_precomp`, `radii`, `sh`, `sh_p`, `geomBuffer`, `binningBuffer`, `imgBuffer`, `phase_offset`, `dc_offset` [S1, unofficial].

**Outputs:**
- `color`: Rendered color map.
- `phasor`: Rendered phasor map.
- `depth`: Rendered depth map.
- `normal`: Rendered normal map.
- `acc`: Accumulated alpha.
- `entropy`: Entropy map.
- `depth_distortion`: Depth distortion map.
- `amp_distortion`: Amplitude distortion map.
- `pixels`: Pixel indices.
- `distribution`: Distribution map.
- `radii`: Radii of visible Gaussians [S1, unofficial].

**Pitfalls:**
- Debug mode incurs significant overhead due to CPU copying [S1, unofficial].
- Phase/DC offsets must be handled correctly based on optimization flags to avoid gradient issues [S1, unofficial].

## Operating characteristics
Time and memory complexity are dominated by the CUDA rasterization kernel, which is $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels, but optimized via binning and sorting. The method supports differentiable rendering of ToF data, enabling joint optimization of geometry and ToF parameters. Strengths include accurate simulation of ToF sensor noise and phase shifts. Limitations include increased memory usage due to phasor attributes and potential precision issues with phase wrapping.

## Background
Originating from the "Gaussian Avatar Reconstruction" and "Gaussian Splatting" literature, this method extends 3D Gaussian Splatting to Time-of-Flight imaging. It was introduced in the context of ToF Gaussian Rasterization (ToFGR) to enable differentiable rendering of depth and amplitude. The implementation is based on the `brownvc/gftorf` repository [S1, unofficial].

## Sources
[S1] brownvc/gftorf, https://github.com/brownvc/gftorf, fetched 2026-10-27 (unofficial)
