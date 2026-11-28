# Epipolar Sampling

Also called: Epipolar Line Sampling, Epipolar Light Shaft Sampling

## Overview
Epipolar sampling is a technique that places sample points along epipolar lines (rays emanating radially from a light source's projection on the screen or feature correspondences between views) to exploit the smooth variation of inscattered light or cross-view features along these lines [S2, S1, unofficial]. This reduces the number of expensive ray-marching or feature-interpolation steps required compared to uniform sampling [S2, unofficial]. In volume rendering, it optimizes light shaft computation by leveraging the observation that light intensity varies smoothly orthogonally to epipolar rays [S2, unofficial]. In multi-view reconstruction, it enables cross-view feature aggregation without full 3D volume rendering by correlating features along epipolar lines [S1, unofficial].

## How it works
The method varies significantly depending on the application domain (light scattering vs. multi-view feature aggregation). The following describes the two primary implementations found in the sources.

### Variant 1: Light Scattering (Screen-Space Epipolar Sampling)
This variant optimizes the computation of inscattered light (light shafts) in participating media [S2, unofficial].

**Inputs:**
- Screen resolution.
- Projected position of the light source on the screen $(L_x, L_y)$.
- Depth buffer (z-buffer) of the scene.
- Number of initial ray-marching samples ($N_{init}$) and interpolation samples ($N_{interp}$) per line [S2, unofficial].

**Outputs:**
- Ins scattered light intensity for each screen pixel.
- Auxiliary textures storing sample indices and coordinates [S2, unofficial].

**Data Structures:**
1. **Epipolar Texture:** A 2D texture where each row corresponds to one epipolar line and each column to a sample location on that line [S2, unofficial].
2. **Auxiliary Integer Texture:** A two-channel texture storing the indices of the two closest ray-marching samples from the same slice for each interpolation sample [S2, unofficial].

**Steps:**

1. **Epipolar Line Generation:**
   - Determine the entry point $E$ as the projected light position on the screen. If the light is behind the camera, $E$ is the point at infinity where rays converge [S2, unofficial].
   - If the light is on the screen:
     - Equidistantly place a user-defined number of exit points along the screen border [S2, unofficial].
     - Connect $E$ to each exit point to form epipolar lines [S2, unofficial].
   - If the light is outside the screen:
     - Cull lines completely outside the visible area [S2, unofficial].
     - Truncate remaining lines against screen borders [S2, unofficial].
   - **Boundary Adjustment:** If the light is close to the screen boundary, rescale epipolar lines by advancing the exit point to strive for a 1:1 correspondence between samples and screen pixels, reducing redundant computations on short lines [S2, unofficial].

2. **Sample Placement:**
   - Place $N_{init}$ initial ray-marching samples equidistantly along each epipolar line to capture low-frequency variations [S2, unofficial].
   - Detect depth breaks (discontinuities) in the depth buffer along each epipolar line [S2, unofficial].
   - Place additional ray-marching samples directly before and after each depth break to capture high-frequency details [S2, unofficial].
   - Fill the remaining positions with interpolation samples [S2, unofficial].

3. **Ray Marching:**
   - Perform expensive ray marching only at the initial and depth-break samples to compute inscattered light intensity [S2, unofficial].

4. **Interpolation:**
   - For interpolation samples, compute intensity by linearly interpolating between the intensities of the two closest ray-marching samples [S2, unofficial].
   - Use the auxiliary integer texture to retrieve the indices of the bounding ray-marching samples for efficient lookup [S2, unofficial].

5. **Pixel Computation:**
   - For each screen pixel, cast an epipolar line through the pixel and compute its location in the epipolar texture [S2, unofficial].
   - Perform bilateral filtering on the gathered inscattered light values [S2, unofficial].
   - If no appropriate samples exist for filtering, mark the sample in a stencil and perform an additional ray-marching pass with fewer steps [S2, unofficial].

### Variant 2: Multi-View Feature Aggregation (Epipolar Encoder)
This variant samples features from other views along epipolar lines for a given ray, enabling cross-view feature aggregation [S1, unofficial].

**Inputs:**
- Image rays (origins, directions) for each pixel in the source view [S1, unofficial].
- Camera extrinsics and intrinsics for all views [S1, unofficial].
- Number of samples per epipolar line segment ($s$) [S1, unofficial].

**Outputs:**
- Aggregated features from other views [S1, unofficial].

**Data Structures:**
- `EpipolarSampling` dataclass containing:
  - `features`: Float Tensor of shape `(batch, view, other_view, ray, sample, channel)` [S1, unofficial].
  - `valid`: Bool Tensor of shape `(batch, view, other_view, ray)` [S1, unofficial].
  - `xy_ray`: Float Tensor of shape `(batch, view, ray, 2)` [S1, unofficial].
  - `xy_sample`: Float Tensor of shape `(batch, view, other_view, ray, sample, 2)` [S1, unofficial].
  - `origins`: Float Tensor of shape `(batch, view, ray, 3)` [S1, unofficial].
  - `directions`: Float Tensor of shape `(batch, view, ray, 3)` [S1, unofficial].

**Steps:**

1. **Ray Generation:**
   - Generate image rays for each pixel in the source view using `sample_image_grid` and `get_world_rays` [S1, unofficial].

2. **Projection:**
   - Project rays onto all other views using `project_rays` with extrinsics, intrinsics, and near/far planes [S1, unofficial].
   - Obtain `xy_min`, `xy_max`, and `overlaps_image` masks [S1, unofficial].
   - Clamp `xy_min` and `xy_max` to zero for infinite values and mask with `overlaps_image` [S1, unofficial].

3. **Sample Depth Generation:**
   - Compute sample depths linearly spaced between 0 and 1:
     $$ \text{sample\_depth} = \frac{\text{torch.arange}(s) + 0.5}{s} $$
     [S1, unofficial]

4. **Coordinate Interpolation:**
   - Interpolate sample coordinates in the target view:
     $$ \text{xy\_sample} = \text{xy\_min} + \text{sample\_depth} \times (\text{xy\_max} - \text{xy\_min}) $$
     [S1, unofficial]

5. **Feature Sampling:**
   - Transpose sample coordinates so the 'view' dimension refers to the target view [S1, unofficial].
   - Exclude the diagonal (self-view) using precomputed indices [S1, unofficial].
   - Sample image features at `xy_sample` coordinates using bilinear `grid_sample` with zero padding [S1, unofficial].

## Subtopics

### [Epipolar Encoder](6426-epipolar-encoder.md)
The Epipolar Encoder is a two-view (or multi-view) neural module that resolves scale ambiguity in 3D reconstruction by aggregating cross-view features along epipolar lines. It solves the problem where camera poses are scaled by an arbitrary factor $s_i$, preventing the inference of metric depth from a single view. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths. This allows the encoder to memorize the correct depth consistent with the scene's arbitrary scale, which is then used for subsequent Gaussian parameter prediction.

### Spot Light Rectification
An extension of the rectification mapping for perspective cameras and local spot lights [S3]. The $\gamma$ coordinate is set to the angle the light ray makes with the direction from the light to the camera [S3]. This results in nonuniform sampling density along the camera ray, causing the $ds/d\gamma$ term in the integral $I(\gamma, \beta)$ to vary with both $\gamma$ and $\beta$, unlike directional lights where it varies only with $\beta$ [S3].

## Operating characteristics
- **Light Scattering:** Dramatically reduces computation time by minimizing expensive ray-marching steps [S2, unofficial]. Rescaling epipolar lines reduces redundant computations on short lines and results in a more natural circular-shaped distribution of samples [S2, unofficial].
- **Multi-View Aggregation:** Enables cross-view feature aggregation without full 3D volume rendering during the encoding phase [S1, unofficial].
- **Limitations:** If the light source is close to the screen boundary, epipolar line lengths vary significantly, causing dense sampling on short lines and redundant computations if not rescaled [S2, unofficial].

## Background
Epipolar sampling for light scattering was proposed by Engelhardt and Dachsbacher to optimize the rendering of light shafts by exploiting the smooth variation of inscattered light along epipolar lines [S2, unofficial]. The application of epipolar sampling to multi-view feature aggregation was introduced in pixelSplat to resolve scale ambiguity in 3D Gaussian Splatting from sparse views [S1, unofficial].

## Sources
[S1] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
[S2] High-Performance Light Scattering Rendering, https://www.scribd.com/document/748135031/822368Yusov-Egor-Practical-Implementation-of-Light, fetched 2026-10-29 (unofficial)
[S3] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
