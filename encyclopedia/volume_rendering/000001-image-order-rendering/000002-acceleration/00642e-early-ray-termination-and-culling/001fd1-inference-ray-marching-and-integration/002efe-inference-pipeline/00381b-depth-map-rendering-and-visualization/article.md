# Depth Map Rendering and Visualization

## Overview
Renders a set of views from a trained dense point cloud (specifically Gaussian Splatting models), prunes points occluded by or too close to the camera, and saves rendered images, depth maps, and colored depth maps [S1, unofficial]. It differs from the parent [Inference Pipeline](5924-inference-pipeline.md) by focusing specifically on the extraction and visualization of depth data and the pruning of invalid geometry, rather than the full evaluation loop including FPS calculation and video generation [S1, unofficial].

## How it works
**Inputs:**
- `gaussians`: Trained Gaussian model containing 3D positions (`get_xyz`), covariances, colors, and opacities [S1, unofficial].
- `views`: List of camera view objects containing intrinsic parameters, extrinsic pose, and image dimensions [S1, unofficial].
- `near`: Scalar distance threshold for pruning points near the camera [S1, unofficial].
- `model_path`, `name`, `iteration`: Path components and iteration index for output directory naming [S1, unofficial].
- `pipeline`, `background`: Rendering pipeline configuration and background color tensor [S1, unofficial].

**Outputs:**
- Rendered color images (`.png`) [S1, unofficial].
- Normalized inverted depth maps (`.png`) [S1, unofficial].
- Colored depth maps (`.png`) [S1, unofficial].
- Video summaries via `ffmpeg` [S1, unofficial].

**Data Structures:**
- `mask_near`: Accumulated tensor of shape `(N,)` or `(N, 1)` where `N` is the number of Gaussians, indicating points within the `near` distance threshold across processed views [S1, unofficial].
- `render_pkg`: Dictionary returned by the renderer, containing keys `"render"` (color image), `"depth"` (raw depth), and `"alpha"` (accumulated alpha) [S1, unofficial].

**Steps:**
1. Create the output directory using the provided path components and iteration index [S1, unofficial].
2. Initialize `mask_near` to `None` [G].
3. Iterate over each view to compute and accumulate a proximity mask: calculate the Euclidean distance between each Gaussian center and the camera center, compare against `near`, and accumulate the boolean mask using addition [S1, unofficial].
4. Prune Gaussians marked in `mask_near` from the model using the inference pruning function [S1, unofficial].
5. For each view, render the scene without gradient tracking, extracting the color image, raw depth, and alpha channel [S1, unofficial].
6. Normalize the raw depth to $[0, 1]$ using the view's local minimum and maximum, then add a background term weighted by alpha: $\text{depth} = \frac{d - d_{\min}}{d_{\max} - d_{\min}} + 1 \cdot (1 - \alpha)$ [S1, unofficial].
7. Save the rendered color image as a PNG [S1, unofficial].
8. Save the inverted normalized depth map as a PNG [S1, unofficial].
9. Convert the depth map to a NumPy array, apply a logarithmic curve function $\text{curve\_fn}(x) = -\ln(x + \epsilon)$ where $\epsilon = \text{np.finfo(np.float32).eps}$, and apply the 'turbo' colormap [S1, unofficial].
10. Convert the colored depth map back to a tensor, permute channels to $(C, H, W)$, and save as a PNG [S1, unofficial].
11. Generate video summaries from the saved image sequences using `ffmpeg` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import os
import numpy as np
import torch
import torchvision.utils
import matplotlib.cm as cm

def render_depth_map(gaussians, views, near, model_path, name, iteration, pipeline, background):
    # 1. Initialize output directory
    render_path = os.path.join(model_path, name, "ours_{}".format(iteration))
    os.makedirs(render_path, exist_ok=True)

    # 2. Initialize mask for points near the camera
    mask_near = None

    # 3. Compute and accumulate proximity mask across views
    for view in views:
        xyz = gaussians.get_xyz  # Shape: (N, 3)
        cam_center = view.camera_center  # Shape: (3,)
        dist = torch.norm(xyz - cam_center.repeat(xyz.shape[0], 1), dim=1, keepdim=True)
        mask_temp = dist < near
        if mask_near is None:
            mask_near = mask_temp
        else:
            mask_near = mask_near + mask_temp

    # 4. Prune points near the camera
    gaussians.prune_points_inference(mask_near)

    # 5. Render each view and save outputs
    for idx, view in enumerate(views):
        # 6. Render without gradient tracking
        with torch.no_grad():
            render_pkg = render(view, gaussians, pipeline, background, inference=True)

        # 7. Extract rendering and depth
        rendering = render_pkg["render"]
        depth_raw = render_pkg['depth']
        alpha = render_pkg["alpha"]

        # 8. Normalize depth and combine with alpha for background
        depth_min = torch.min(depth_raw)
        depth_max = torch.max(depth_raw)
        depth_norm = (depth_raw - depth_min) / (depth_max - depth_min)
        depth = depth_norm + 1.0 * (1.0 - alpha)

        # 9. Save rendered image
        torchvision.utils.save_image(rendering, os.path.join(render_path, '{0:05d}'.format(idx) + ".png"))

        # 10. Save inverted depth map
        torchvision.utils.save_image(1 - depth, os.path.join(render_path, 'depth_{0:05d}'.format(idx) + ".png"))

        # 11. Generate colored depth map
        depth_est = depth.squeeze().cpu().numpy()
        eps = np.finfo(np.float32).eps
        depth_curve_fn = lambda x: -np.log(x + eps)
        # Applies colormap with curve function [G]
        depth_est = visualize_cmap(depth_est, np.ones_like(depth_est), cm.get_cmap('turbo'), curve_fn=depth_curve_fn).copy()

        # 12. Save colored depth map
        depth_est_tensor = torch.as_tensor(depth_est).permute(2, 0, 1)
        torchvision.utils.save_image(depth_est_tensor, os.path.join(render_path, 'cdepth_{0:05d}'.format(idx) + ".png"))

    # 13. Generate video outputs using ffmpeg
    # (Standard ffmpeg command generation omitted)
```

**Parameters:**
- `near`: Distance threshold for pruning points near the camera. Value not given in the sources.

**Edge Cases and Pitfalls:**
- **Depth Normalization:** The depth normalization uses global min/max of the rendered depth map, not the scene bounds. This can cause inconsistent depth scaling across frames if the visible depth range varies significantly [S1, unofficial].
- **Background Handling:** The formula adds $1 \cdot (1 - \alpha)$, setting background depth to $1.0$ (normalized). If foreground depth also normalizes near $1.0$, depth and background may be indistinguishable in the raw depth map [S1, unofficial].
- **Mask Accumulation:** Using addition for mask accumulation creates integer tensors. The pruning function must handle this correctly by treating any positive value as a prune flag [S1, unofficial].

## Subtopics

### [4DGS Rendering Pipeline](16175-4dgs-rendering-pipeline.md)
The 4DGS Rendering Pipeline is the inference-stage script that iterates over a sequence of camera views to render images using the trained 4D Gaussian Splatting model, saves the ground truth and rendered images, and generates a video summary. It differs from the parent [Depth Map Rendering and Visualization](8160-depth-map-rendering-and-visualization.md) by focusing specifically on the full image-order rendering loop for evaluation and visualization, including FPS calculation and video generation, rather than just depth map extraction and pruning.

### [Double-sided Depth Rendering](18756-double-sided-depth-rendering.md)
Double-sided depth rendering generates a complete depth map for a mesh by rendering front-facing and back-facing geometry separately and combining the results. It solves the problem of missing depth information for occluded or back-facing surfaces that standard single-pass rendering culls. Unlike [4DGS Rendering Pipeline](16175-4dgs-rendering-pipeline.md) or [Open3D Image Rendering Pipeline](19719-open3d-image-rendering-pipeline.md) which may rely on specific primitive types or culling flags, this method explicitly handles non-convex geometry by ensuring every ray intersection is captured.

### [Open3D Image Rendering Pipeline](19719-open3d-image-rendering-pipeline.md)
This method converts raw rendering outputs (depth, opacity, ellipsoid parameters, or RGB images) into Open3D-compatible geometry images for visualization, handling different modalities via UI checkboxes. It sets itself apart from sibling pipelines by directly interfacing with OpenGL for ellipsoid rendering and using `imgviz` for depth/opacity normalization, whereas siblings like the 4DGS pipeline may use different backend renderers or data structures.

### [Hierarchical Depth Map Rendering](20161-hierarchical-depth-map-rendering.md)
Hierarchical Depth Map Rendering is an image-based rendering technique that combines a hierarchical geometric model with precomputed multi-layer depth images to efficiently render complex vegetation. It differs from sibling methods by resampling precomputed multi-layer z-buffer images of hierarchical plant subparts rather than rendering geometry directly or using generic point clouds. The method is designed for offline or precomputed scenarios, as it is not real-time.

### Flow Data Depth Rendering
Converts optical flow and camera intrinsics/extrinsics into a depth map [S2, unofficial]. It handles validity, mesh export, and depth padding via morphological operations [S2, unofficial].
1. **Calculate Offset:** $\text{offset} = \text{data}['\text{ref\_intr}'][:, 0, 2] - \text{data}['\text{intr}'][:, 0, 2]$ [S2, unofficial].
2. **Compute Disparity:** $\text{disparity} = \text{offset} - \text{data}['\text{flow\_pred}']$ [S2, unofficial].
3. **Compute Depth:** $\text{depth} = -\frac{\text{disparity}}{\text{data}['\text{Tf\_x}']}$ [S2, unofficial].
4. **Mask Invalid Regions:** Multiply depth by `data['mask']` to zero out invalid regions [S2, unofficial].
5. **Export Mesh (Optional):** If `pts_out_path` is provided, export depth mesh using `depth2mesh` [S2, unofficial].
6. **Transform to Original Camera Space:** Transform points to original camera space using `perspective` with `calib_ori = intr_ori @ extr_ori`. Invert z-depth [S2, unofficial].
7. **Render Perspective Depth:** Render perspective depth using `taichi_render.render_perspective_depth` with valid mask [S2, unofficial].
8. **Pad Depth:** Apply padding to `render_depth` using `taichi_render.pad_depth` and store in `data['lmain']['depth_ori']` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Gaussians and views. Pruning is $O(N)$ where $N$ is the number of Gaussians. Rendering is $O(N \cdot V)$ where $V$ is the number of views [G].
- **Memory Complexity:** $O(N)$ for storing Gaussian attributes. The mask accumulation requires $O(N)$ storage [G].
- **Strengths:** Efficiently removes invalid geometry (points too close to camera) that can cause artifacts. Provides standardized depth visualization for evaluation [S1, unofficial].
- **Limitations:** Depth normalization is local to each view, not global. Pruning is aggressive and irreversible [S1, unofficial].
- **When to use:** When evaluating depth accuracy or visualizing depth maps from Gaussian Splatting models [S1, unofficial].
- **When not to use:** When global depth consistency across views is required without post-processing [G].

## Background
Originated in the context of 3D Gaussian Splatting inference pipelines for novel view synthesis evaluation [S1, unofficial]. It is used to generate evaluation metrics and visualizations for trained models [S1, unofficial].

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
[S2] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
