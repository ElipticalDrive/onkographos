# Depth Rendering with Constant Opacity

Also called: Depth-Only Rendering Mode

## Overview
Depth Rendering with Constant Opacity is a specialized rendering pass for 3D Gaussian Splatting designed to extract clean, geometrically accurate depth maps by decoupling depth accumulation from opacity weighting [S1, unofficial]. Unlike standard rendering where depth is influenced by the varying opacity of Gaussians, this method forces uniform opacity and color across all Gaussians, ensuring that the rasterizer's depth accumulation is driven solely by geometric distance [S1, unofficial]. It serves as a foundation for variants like Normalized Depth Rendering and Depth Rendering with Variable Opacity.

## How it works
**Inputs:**
- `pc`: The Gaussian model object containing 3D means, scales, rotations, covariances, and SH coefficients.
- `viewpoint_camera`: The camera object containing field of view (FoV) and projection matrices.
- `image_width`, `image_height`: The resolution of the output depth map.
- `value`: The constant opacity multiplier (default 0.95) [S1, unofficial].

**Outputs:**
- `render`: The rendered image (all ones due to constant color).
- `depth`: The accumulated depth buffer.
- `alpha`: The accumulated opacity buffer.
- `viewspace_points`: The 3D positions of the Gaussians in view space.
- `visibility_filter`: A boolean mask indicating which Gaussians contributed to the image (radii > 0).
- `radii`: The radius of each Gaussian's bounding box in screen space.

**Data Structures:**
- `screenspace_points`: A tensor of shape `(num_gaussians, 2)` initialized to zeros, with gradient retention enabled to allow backpropagation if needed for training [S1, unofficial].
- `opacity`: A tensor of shape `(num_gaussians, 1)` filled with the constant `value` [S1, unofficial].
- `colors_precomp`: A tensor of shape `(num_gaussians, 3)` filled with ones, removing color information [S1, unofficial].
- `shs`: Set to `None` to disable spherical harmonics processing [S1, unofficial].

**Steps:**
1. **Initialize Screenspace Points:** Create a tensor `screenspace_points` with the same dtype and device as `pc.get_xyz`, initialized to zero. Enable gradient retention:
   ```python
   screenspace_points = torch.zeros_like(pc.get_xyz, dtype=pc.get_xyz.dtype, requires_grad=True, device="cuda") + 0
   try:
       screenspace_points.retain_grad()
   except:
       pass
   ```
   [S1, unofficial]

2. **Compute Tangent FOV:** Calculate the tangent of half the field of view from the `viewpoint_camera`:
   $$ \text{tanfovx} = \tan(\text{FoV}_x / 2) $$
   $$ \text{tanfovy} = \tan(\text{FoV}_y / 2) $$
   [S1, unofficial]

3. **Configure Rasterization Settings:** Create a `GaussianRasterizationSettings` object with:
   - Image dimensions: `(image_width, image_height)`
   - Tangent FOV: `(tanfovx, tanfovy)`
   - Background color: Typically white `[1, 1, 1]` or black `[0, 0, 0]` depending on the dataset [S1, unofficial].
   - Scale modifier: As defined in the pipeline.
   - View and projection matrices: From `viewpoint_camera`.
   - SH degree: From the pipeline.
   - Camera position: From `viewpoint_camera`.
   - Prefiltered: `False` [S1, unofficial].

4. **Set Gaussian Attributes:**
   - `means3D`: Set to `pc.get_xyz`.
   - `means2D`: Set to `screenspace_points`.
   - `opacity`: Set to `torch.ones(num_gaussians, 1, device=device) * value` [S1, unofficial].
   - `colors_precomp`: Set to `torch.ones_like(pc.get_xyz)` [S1, unofficial].
   - `shs`: Set to `None` [S1, unofficial].

5. **Detach Scales and Rotations:** Within a `torch.no_grad()` block:
   - Set `scales`, `rotations`, and `cov3D_precomp` to `None` initially.
   - If `pipe.compute_cov3D_python` is true, compute `cov3D_precomp` using `pc.get_covariance(scaling_modifier)`.
   - Otherwise, set `scales = pc.get_scaling.detach()` and `rotations = pc.get_rotation.detach()` [S1, unofficial].
   - This step prevents gradient leakage into depth estimation during training [S1, unofficial].

6. **Rasterize:** Call the Gaussian rasterizer with the configured attributes:
   - Inputs: `means3D`, `means2D`, `shs=None`, `colors_precomp`, `opacity`, `scales`, `rotations`, `cov3D_precomp`.
   - Outputs: `rendered_image`, `radii`, `rendered_depth`, `rendered_alpha` [S1, unofficial].

7. **Return Results:** Return a dictionary containing:
   - `render`: `rendered_image`
   - `depth`: `rendered_depth`
   - `alpha`: `rendered_alpha`
   - `viewspace_points`: The 3D points in view space.
   - `visibility_filter`: `radii > 0`
   - `radii`: The screen-space radii [S1, unofficial].

**Parameters:**
- `value`: The constant opacity multiplier. Default is 0.95 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Gradient Leakage:** Failing to detach scales and rotations or disable SHs can cause gradients to leak into the depth estimation, corrupting the geometric accuracy [S1, unofficial].
- **Precision:** The use of `torch.zeros_like` with explicit device placement ensures no NaNs or infinities are introduced during initialization [S1, unofficial].
- **Color Information:** Setting `colors_precomp` to ones and `shs` to `None` ensures that no color information affects the depth pass, which is critical for pure depth extraction [S1, unofficial].

## Subtopics

### [Normalized Depth Rendering](8298-normalized-depth-rendering.md)
Computes a depth map by dividing the accumulated depth buffer by the accumulated opacity buffer, correcting for transparency artifacts inherent in standard Gaussian splatting compositing. Unlike its parent method, which forces constant opacity to decouple depth from alpha weighting, this method preserves the original scene's opacity distribution while producing a geometrically accurate depth estimate.

### [Depth Rendering with Variable Opacity](12622-depth-rendering-with-variable-opacity.md)
A feature that enables Gaussian splatting backends to write a per-pixel depth buffer (NDC depth, +inf where there is no surface) which can be read via getGaussianSplattingDepth() to composite other geometry into the splat scene.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Gaussians and pixels, as it involves a single rasterization pass [S1, unofficial].
- **Memory Complexity:** Linear with respect to the number of Gaussians for storing intermediate tensors like `screenspace_points` and `opacity` [S1, unofficial].
- **Performance:** The method is efficient as it avoids complex shading calculations by using constant color and opacity [S1, unofficial].
- **Strengths:** Produces clean, geometrically accurate depth maps without shading or opacity variations [S1, unofficial].
- **Limitations:** Does not provide color information or opacity variations, making it unsuitable for standard image rendering [S1, unofficial].
- **When to Use:** When a pure depth map is required for downstream tasks like geometry reconstruction or compositing [S1, unofficial].
- **When Not to Use:** When color or opacity information is needed [S1, unofficial].

## Background
Depth Rendering with Constant Opacity is a technique developed within the context of 3D Gaussian Splatting to address the need for accurate depth extraction. It was introduced in the DNGaussian repository, which provides unofficial implementations and extensions to the original Gaussian Splatting framework [S1, unofficial]. The method is particularly useful in applications where geometric accuracy is paramount, such as in neural rendering and 3D reconstruction pipelines.

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
