# Depth Map Rendering and Visualization


## Overview
Renders a set of views from a trained dense point cloud (specifically Gaussian Splatting models), prunes points occluded by or too close to the camera, and saves rendered images, depth maps, and colored depth maps [S1, unofficial]. It differs from the parent [Inference Pipeline](5924-inference-pipeline.md) by focusing specifically on the extraction and visualization of depth data and the pruning of invalid geometry, rather than the full evaluation loop including FPS calculation and video generation [S1, unofficial].

## How it works
**Inputs:**
- `gaussians`: The trained Gaussian model containing 3D positions (`xyz`), covariances, colors, and opacities.
- `view`: A camera view object containing intrinsic parameters, extrinsic pose, and image dimensions.
- `near`: A scalar distance threshold for pruning points near the camera.
- `render_path`: Directory path for output images.
- `iteration`: Current training/inference iteration index (used for naming).

**Outputs:**
- Rendered color images (`.png`).
- Normalized depth maps (`.png`).
- Colored depth maps (`.png`).
- Video summaries (via ffmpeg).

**Data Structures:**
- `mask_near`: A boolean tensor of shape `(N,)` where `N` is the number of Gaussians, indicating points within the `near` distance threshold.
- `render_pkg`: A dictionary returned by the renderer, containing at least keys `"render"` (color image), `"depth"` (raw depth), and `"alpha"` (accumulated alpha).

**Steps:**

1. **Initialize Output Directory:**
   Create the output directory using the model path, name, and iteration index:
   ```python
   render_path = os.path.join(model_path, name, "ours_{}".format(iteration))
   makedirs(render_path, exist_ok=True)
   ```

2. **Compute Proximity Mask:**
   Calculate the Euclidean distance between each Gaussian center and the camera center. Identify Gaussians closer than `near`.
   Let $P$ be the set of Gaussian centers with shape $(N, 3)$ and $C$ be the camera center with shape $(3,)$.
   $$ D_i = \| P_i - C \|_2 $$
   The mask for the current view is:
   $$ M_i = \begin{cases} 1 & \text{if } D_i < \text{near} \\ 0 & \text{otherwise} \end{cases} $$
   Accumulate this mask across all views processed so far to prune points that are near the camera in *any* view:
   ```python
   mask_temp = (gaussians.get_xyz - view.camera_center.repeat(gaussians.get_xyz.shape[0], 1)).norm(dim=1, keepdim=True) < near
   if mask_near is None:
       mask_near = mask_temp
   else:
       mask_near = mask_near + mask_temp
   ```
   *Note: The source uses addition (`+`) rather than logical OR (`|`). In boolean contexts, this may result in values > 1. The pruning function likely treats any non-zero value as True.*

3. **Prune Points:**
   Remove Gaussians marked in `mask_near` from the model.
   ```python
   gaussians.prune_points_inference(mask_near)
   ```

4. **Render View:**
   For each view, perform rendering without gradient tracking.
   ```python
   with torch.no_grad():
       render_pkg = render(view, gaussians, pipeline, background, inference=True)
   ```

5. **Extract and Normalize Depth:**
   Extract raw depth and alpha from `render_pkg`. Normalize the depth to $[0, 1]$ and combine with alpha to handle background.
   Let $d$ be `render_pkg['depth']` and $\alpha$ be `render_pkg['alpha']`.
   $$ d_{min} = \min(d), \quad d_{max} = \max(d) $$
   $$ d_{norm} = \frac{d - d_{min}}{d_{max} - d_{min}} $$
   The final depth map is computed as:
   $$ \text{depth} = d_{norm} + 1 \cdot (1 - \alpha) $$
   *Note: This formula adds a value of 1 for pixels where alpha is 0 (background), effectively pushing background depth to the maximum normalized value.*

6. **Save Rendered Image:**
   Save the color image.
   ```python
   torchvision.utils.save_image(rendering, os.path.join(render_path, '{0:05d}'.format(idx) + ".png"))
   ```

7. **Save Inverted Depth Map:**
   Save the inverted normalized depth.
   ```python
   torchvision.utils.save_image(1 - depth, os.path.join(render_path, 'depth_{0:05d}'.format(idx) + ".png"))
   ```

8. **Generate Colored Depth Map:**
   Convert depth to numpy, apply a colormap, and save.
   ```python
   depth_est = depth.squeeze().cpu().numpy()
   # Apply colormap 'turbo' with a logarithmic curve function
   depth_est = visualize_cmap(depth_est, np.ones_like(depth_est), cm.get_cmap('turbo'), curve_fn=depth_curve_fn).copy()
   ```
   The curve function is defined as:
   $$ \text{curve\_fn}(x) = -\ln(x + \epsilon) $$
   where $\epsilon = \text{np.finfo(np.float32).eps}$.

9. **Save Colored Depth Map:**
   Convert back to tensor, permute channels, and save.
   ```python
   depth_est = torch.as_tensor(depth_est).permute(2, 0, 1)
   torchvision.utils.save_image(depth_est, os.path.join(render_path, 'cdepth_{0:05d}'.format(idx) + ".png"))
   ```

10. **Generate Video:**
    Use ffmpeg to create video summaries from the saved render, depth, and colored depth images.

**Parameters:**
- `near`: Distance threshold for pruning points near the camera. Value not given in the sources.

**Edge Cases and Pitfalls:**
- **Depth Normalization:** The depth normalization uses global min/max of the *rendered* depth map, not the scene bounds. This can cause inconsistent depth scaling across frames if the visible depth range varies significantly.
- **Background Handling:** The formula $d_{norm} + (1 - \alpha)$ sets background depth to 1.0 (normalized). If the foreground depth also normalizes to near 1.0, depth and background may be indistinguishable in the raw depth map.
- **Mask Accumulation:** Using addition for mask accumulation (`mask_near + mask_temp`) creates integer tensors. The pruning function must handle this correctly (treating any positive value as "prune").

## Subtopics

### [4DGS Rendering Pipeline](16175-4dgs-rendering-pipeline.md)
The 4DGS Rendering Pipeline is the inference-stage script that iterates over a sequence of camera views to render images using the trained 4D Gaussian Splatting model, saves the ground truth and rendered images, and generates a video summary. It differs from the parent [Depth Map Rendering and Visualization](8160-depth-map-rendering-and-visualization.md) by focusing specifically on the full image-order rendering loop for evaluation and visualization, including FPS calculation and video generation, rather than just depth map extraction and pruning.

### Flow Data Depth Rendering
Converts optical flow and camera intrinsics/extrinsics into a depth map [S2, unofficial]. It handles validity, mesh export, and depth padding via morphological operations [S2, unofficial].

**Steps:**
1. **Calculate Offset:**
   $$ \text{offset} = \text{data}['\text{ref\_intr}'][:, 0, 2] - \text{data}['\text{intr}'][:, 0, 2] $$
2. **Compute Disparity:**
   $$ \text{disparity} = \text{offset} - \text{data}['\text{flow\_pred}'] $$
3. **Compute Depth:**
   $$ \text{depth} = -\frac{\text{disparity}}{\text{data}['\text{Tf\_x}']} $$
4. **Mask Invalid Regions:**
   Multiply depth by `data['mask']` to zero out invalid regions.
5. **Export Mesh (Optional):**
   If `pts_out_path` is provided, export depth mesh using `depth2mesh`.
6. **Transform to Original Camera Space:**
   Transform points to original camera space using `perspective` with `calib_ori = intr_ori @ extr_ori`. Invert z-depth.
7. **Render Perspective Depth:**
   Render perspective depth using `taichi_render.render_perspective_depth` with valid mask.
8. **Pad Depth:**
   Apply padding to `render_depth` using `taichi_render.pad_depth` and store in `data['lmain']['depth_ori']`.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Gaussians and views. Pruning is $O(N)$ where $N$ is the number of Gaussians. Rendering is $O(N \cdot V)$ where $V$ is the number of views.
- **Memory Complexity:** $O(N)$ for storing Gaussian attributes. The mask accumulation requires $O(N)$ storage.
- **Strengths:** Efficiently removes invalid geometry (points too close to camera) that can cause artifacts. Provides standardized depth visualization for evaluation.
- **Limitations:** Depth normalization is local to each view, not global. Pruning is aggressive and irreversible.
- **When to use:** When evaluating depth accuracy or visualizing depth maps from Gaussian Splatting models.
- **When not to use:** When global depth consistency across views is required without post-processing.

## Background
Originated in the context of 3D Gaussian Splatting inference pipelines, specifically within the DNGaussian project [S1, unofficial]. It is used to generate evaluation metrics and visualizations for novel view synthesis.

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
[S2] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
