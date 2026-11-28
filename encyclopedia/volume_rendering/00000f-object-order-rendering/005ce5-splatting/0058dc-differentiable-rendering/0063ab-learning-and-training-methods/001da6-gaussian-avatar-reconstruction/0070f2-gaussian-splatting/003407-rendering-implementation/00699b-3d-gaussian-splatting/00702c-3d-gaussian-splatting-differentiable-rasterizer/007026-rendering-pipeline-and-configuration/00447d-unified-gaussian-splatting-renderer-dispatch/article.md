# Unified Gaussian Splatting Renderer Dispatch

Also called: Gaussian Splatting Renderer Dispatch, render function dispatch, vggt Gaussian Renderer

## Overview
The Unified Gaussian Splatting Renderer Dispatch is a routing mechanism that selects and invokes the appropriate differentiable Gaussian rasterization backend (3DGS, 2DGS, or GGGS) based on the model's geometric configuration or explicit mode flags [S1, unofficial]. It abstracts the heterogeneity of Gaussian splatting variants by handling the preparation of input tensors (means, opacities, spherical harmonics, covariances) and the post-processing of rasterizer outputs (depth, normals, alpha) into a unified interface [S1, unofficial]. It sets itself apart from sibling methods by acting as the central dispatcher that validates inputs and routes execution, rather than performing the rasterization itself [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A Gaussian point cloud object `pc` containing learned parameters (means, opacities, scales, rotations, spherical harmonics); a `viewpoint_camera` object containing projection matrices and field of view; a `pipe` configuration object controlling computation modes; and an optional `gs_mode` string.
*   **Outputs:** Rendered color image, alpha mask, depth map, and optionally normals, depending on the selected backend and post-processing steps.

**Data Structures:**
*   `pc`: Contains tensors for `means` (3D coordinates), `opacities` (scalar opacity per Gaussian), `scales` (scaling factors), `rotations` (quaternions), and `harmonics` or `features` (spherical harmonic coefficients).
*   `viewpoint_camera`: Contains `FoVx`, `FoVy` (field of view angles), `viewmatrix` (world-to-camera), `projmatrix` (camera-to-clip), `campos` (camera position).
*   `GaussianRasterizationSettings`: A configuration object passed to the rasterizer constructor, containing image dimensions, FOV tangents, background color, scaling modifier, view/projection matrices, SH degree, and camera position.

**Numbered Steps:**

1.  **Extract Batch-Specific Attributes:**
    Identify the active Gaussians for the current batch index `batch_idx`. Extract 3D means `xyz` and unsqueeze opacities to shape `(-1, 1)` [S1, unofficial].
    ```python
    xyz = pc.means[batch_idx]
    opacity = pc.opacities[batch_idx].unsqueeze(-1)
    ```

2.  **Initialize Screen-Space Points:**
    Create a zero-initialized tensor for screen-space points to capture gradients during the backward pass. This tensor must require gradients and reside on the CUDA device [S1, unofficial].
    ```python
    screenspace_points = torch.zeros_like(xyz, dtype=xyz.dtype, requires_grad=True, device="cuda")
    ```

3.  **Compute Tangent Field of View:**
    Calculate the tangent of half the horizontal and vertical field of view angles from the `viewpoint_camera` [S1, unofficial].
    ```python
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    ```

4.  **Determine Render Mode:**
    Select the rasterization backend mode. If `gs_mode` is explicitly provided, use it. Otherwise, infer the mode from the scale dimensions of the point cloud: if the second dimension of `pc.scales[batch_idx]` is 2, select "2DGS"; otherwise, default to "3DGS" [S1, unofficial].

5.  **Select Rasterizer Classes:**
    Map the determined `requested_render_mode` to the corresponding `GaussianRasterizationSettings` and `GaussianRasterizer` classes from the appropriate library:
    *   "2DGS": `diff_surfel_rasterization`
    *   "3DGS": `diff_gaussian_rasterization`
    *   "GGGS": `gggs_diff_gaussian_rasterization` [S1, unofficial]

6.  **Construct GaussianRasterizationSettings:**
    Instantiate the settings object with common parameters: `image_height`, `image_width`, `tanfovx`, `tanfovy`, `bg` (background color), `scaling_modifier`, `viewmatrix`, `projmatrix`, `sh_degree`, `campos`, `prefiltered=False`, and `debug` [S1, unofficial].
    *   For **GGGS**, additionally pass `kernel_size`, `sg_degree=0`, and `require_depth` [S1, unofficial].

7.  **Prepare Covariance:**
    *   If `pipe.compute_cov3D_python` is true: Compute `cov3D_precomp` by transforming the world-space covariance to pixel space using `ndc2pix` and `world2pix` matrices. Reshape the result to `(-1, 9)` in column-major order [S1, unofficial].
    *   Otherwise: Extract `scales` and `rotations` directly from `pc` [S1, unofficial].

8.  **Prepare Colors:**
    *   If `override_color` is None and `pipe.convert_SHs_python` is true:
        1.  Extract spherical harmonic features: `shs_view = pc.get_features.transpose(1, 2).view(-1, 3, (pc.max_sh_degree+1)**2)` [S1, unofficial].
        2.  Compute normalized viewing directions: `dir_pp_normalized = dir_pp / dir_pp.norm(dim=1, keepdim=True)` [S1, unofficial].
        3.  Evaluate SHs to RGB: `sh2rgb = eval_sh(pc.active_sh_degree, shs_view, dir_pp_normalized)` [S1, unofficial].
        4.  Clamp and offset: `colors_precomp = torch.clamp_min(sh2rgb + 0.5, 0.0)` [S1, unofficial].
    *   Else: Set `shs` from `pc.harmonics[batch_idx].permute(0, 2, 1)` [S1, unofficial].

9.  **Execute Rasterizer:**
    Call the selected `rasterizer` with the prepared arguments: `means3D`, `means2D`, `shs`, `colors_precomp`, `opacity`, `scales`, `rotations`, `cov3D_precomp` [S1, unofficial].
    *   For **GGGS**, additionally pass `sg_axis`, `sg_sharpness`, and `sg_color` [S1, unofficial].

10. **Post-Process 2DGS Outputs:**
    If the mode is 2DGS, extract the multi-channel output `allmap` [S1, unofficial]:
    *   `render_alpha = allmap[1:2]`
    *   `render_normal = allmap[2:5]` (transformed to world space)
    *   `render_depth_median = allmap[5:6]`
    *   `render_depth_expected = allmap[0:1] / render_alpha`
    *   `render_dist = allmap[6:7]`
    
    Compute the final surface depth and normal [S1, unofficial]:
    ```python
    surf_depth = render_depth_expected * (1 - pipe.depth_ratio) + pipe.depth_ratio * render_depth_median
    surf_normal = depth_to_normal(viewpoint_camera, surf_depth) * render_alpha.detach()
    ```

11. **Post-Process 3DGS Outputs:**
    If the mode is 3DGS, compute the depth from the inverse depth output `inv_depth` [S1, unofficial]:
    ```python
    depth = 1.0 / inv_depth.clamp(min=1e-6)
    ```

12. **Post-Process GGGS Outputs:**
    If the mode is GGGS, retrieve optional attributes `sg_axis`, `sg_sharpness`, `sg_color` from `pc` (handling defaults and tensor dimensions) [S1, unofficial]. Return `rendered_median_depth`, `rendered_alpha`, and `rendered_normal` [S1, unofficial].

**Parameters with Typical Values:**
*   `prefiltered`: False [S1, unofficial].
*   `sg_degree` (GGGS): 0 [S1, unofficial].
*   `min_depth` (clamp): 1e-6 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Inverse depth calculation requires clamping to `1e-6` to avoid division by zero or numerical instability [S1, unofficial].
*   **Dimensions:** Ensure `opacity` is unsqueezed to `(-1, 1)` to match rasterizer expectations [S1, unofficial].
*   **Mode Inference:** The fallback logic for `gs_mode` relies on the shape of `pc.scales`. If the scale tensor dimension is ambiguous, explicit `gs_mode` should be provided [S1, unofficial].

## Subtopics

### [Gaussian Rasterization Variable Preparation](17689-gaussian-rasterization-variable-preparation.md)
Constructs the specific tensor dictionaries required by differentiable Gaussian rasterization kernels, transforming raw learned parameters into the formats expected by the CUDA/C++ backend. It filters active Gaussians by temporal validity, normalizes geometric attributes, and computes derived values such as opacities and 2D means, ensuring the rasterizer receives consistent, normalized inputs for forward and backward passes.

### [Gaussian Splat View of TSDF Band](19215-gaussian-splat-view-of-tsdf-band.md)
A visualization technique that renders a filtered Truncated Signed Distance Function (TSDF) voxel band as a continuous surface using isotropic Gaussian splats, where one Gaussian is generated per surface voxel. It differs from mesh-based extraction (e.g., marching cubes) by providing a differentiable, continuous representation suitable for real-time interactive visualization, and from raw voxel rendering by smoothing discrete grid artifacts into a coherent surface.

## Operating characteristics
*   **Time Complexity:** Dominated by the CUDA rasterization kernel execution, which is generally $O(N)$ with respect to the number of Gaussians $N$, though overhead from Python-side tensor preparation (SH evaluation, covariance transformation) adds constant factors.
*   **Memory Complexity:** Requires storage for input tensors (means, SHs, covariances) and output buffers (color, depth, alpha, normals). The `screenspace_points` tensor adds $O(N)$ memory for gradient storage [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the number of active Gaussians and image resolution. The dispatch overhead is negligible compared to the rasterization cost.
*   **Strengths:** Unified interface for multiple Gaussian variants (3DGS, 2DGS, GGGS); automatic handling of data preparation and post-processing; supports differentiable rendering for training.
*   **Limitations:** Relies on specific CUDA backends (`diff_gaussian_rasterization`, etc.); Python-side SH evaluation can be a bottleneck if not optimized; mode inference logic may fail if scale dimensions are non-standard.
*   **When to use:** When training or rendering scenes using different Gaussian splatting variants (3D, 2D, or GGGS) and needing a consistent API.
*   **When not to use:** When using a fixed, single variant without need for dispatch abstraction, or when hardware does not support the required CUDA kernels.

## Background
The Unified Gaussian Splatting Renderer Dispatch originates from the VGGT (Video Gaussian Gaussian Transformer) project, specifically within the rendering pipeline of the VGVGGT repository [S1, unofficial]. It was developed to support the integration of various Gaussian splatting techniques (3DGS, 2DGS, GGGS) into a single coherent framework for video-based 3D reconstruction and rendering tasks [S1, unofficial]. The method is part of the broader effort to make Gaussian splatting differentiable and trainable for dynamic scene modeling [S1, unofficial].

## Sources
[S1] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
