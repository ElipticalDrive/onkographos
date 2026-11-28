# 4DGS Rendering Pipeline

Also called: Rendering Pipeline for 4DGS

## Overview
The 4DGS Rendering Pipeline is the inference-stage script that iterates over a sequence of camera views to render images using the trained 4D Gaussian Splatting model, saves the ground truth and rendered images, and generates a video summary [S1, unofficial]. It differs from the parent [Depth Map Rendering and Visualization](8160-depth-map-rendering-and-visualization.md) by focusing specifically on the full image-order rendering loop for evaluation and visualization, including FPS calculation and video generation, rather than just depth map extraction and pruning [S1, unofficial].

## How it works
**Inputs:**
- `views`: A list of camera view objects containing projection matrices, image data, and metadata.
- `gaussians`: The 4D Gaussian splatting model parameters (positions, covariances, opacities, colors, etc.).
- `pipeline`: Rendering configuration parameters (e.g., rasterization settings).
- `background`: Background color tensor.
- `name`: Dataset identifier (e.g., 'train', 'test', or specific scene name).
- `cam_type`: Camera type string (e.g., 'PanopticSports' or standard pinhole).
- `model_path`, `iteration`: Paths to locate output directories.

**Outputs:**
- Rendered images (PNG) in `renders/`.
- Ground truth images (PNG) in `gt/`.
- Video file `video_rgb.mp4`.
- Console output: FPS metric.

**Data Structures:**
- `render_images`: List of HWC uint8 numpy arrays for visualization.
- `render_list`: List of raw tensor renders (CHW format) for saving.
- `gt_list`: List of raw tensor ground truth images (CHW format).

**Steps:**
1. **Setup Paths**: Create directories `renders` and `gt` under `model_path/name/ours_{iteration}/`.
2. **Initialize**: Create empty lists `render_images`, `gt_list`, and `render_list`. Record start time `time1` when processing the first view.
3. **Iterate Views**: Loop through each `view` in `views` using a progress bar.
4. **Render**: Call the core rendering function `render(view, gaussians, pipeline, background, cam_type=cam_type)`. Extract the `'render'` key from the returned dictionary.
5. **Convert and Store**:
   - Convert the raw render tensor to 8-bit uint8 using `to8b` (clip values to [0, 1], multiply by 255, cast to uint8).
   - Transpose the result from CHW to HWC format.
   - Append the HWC array to `render_images`.
   - Append the raw tensor to `render_list`.
6. **Handle Ground Truth**: If `name` is 'train' or 'test':
   - If `cam_type` is not 'PanopticSports', extract `view.original_image[0:3, :, :]`.
   - If `cam_type` is 'PanopticSports', extract `view['image'].cuda()`.
   - Append the GT tensor to `gt_list`.
7. **Calculate FPS**: After the loop, record end time `time2`. Calculate FPS as `(len(views) - 1) / (time2 - time1)`.
8. **Save Images**: Use `multithread_write` to save `gt_list` to the `gt` directory and `render_list` to the `renders` directory. Files are named `{index:05d}.png`.
9. **Create Video**: Use `imageio.mimwrite` to generate `video_rgb.mp4` in the output directory using `render_images` with a frame rate of 36 FPS.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time
import os
import imageio
import numpy as np
import torch

def render_pipeline(views, gaussians, pipeline, background, name, cam_type, model_path, iteration):
    # Setup Paths
    method_dir = os.path.join(model_path, name, f"ours_{iteration}")
    renders_dir = os.path.join(method_dir, "renders")
    gts_dir = os.path.join(method_dir, "gt")
    os.makedirs(renders_dir, exist_ok=True)
    os.makedirs(gts_dir, exist_ok=True)

    render_images = []
    gt_list = []
    render_list = []

    # Iterate Views
    for idx, view in enumerate(views):
        if idx == 0:
            time1 = time.time()
        
        # Render
        # Note: render() is assumed to be defined in the parent context or imported
        rendering = render(view, gaussians, pipeline, background, cam_type=cam_type)["render"]
        
        # Convert and Store
        # to8b clips to [0,1], multiplies by 255, casts to uint8
        render_uint8 = to8b(rendering)
        # Transpose from CHW to HWC
        render_hwc = render_uint8.transpose(1, 2, 0)
        render_images.append(render_hwc)
        render_list.append(rendering)
        
        # Handle Ground Truth
        if name in ["train", "test"]:
            if cam_type != "PanopticSports":
                gt = view.original_image[0:3, :, :]
            else:
                gt = view['image'].cuda()
            gt_list.append(gt)

    time2 = time.time()
    # Calculate FPS
    fps = (len(views) - 1) / (time2 - time1)
    print(f"FPS: {fps}")

    # Save Images
    # multithread_write is a helper function for parallel I/O
    multithread_write(gt_list, gts_dir)
    multithread_write(render_list, renders_dir)

    # Create Video
    video_path = os.path.join(method_dir, "video_rgb.mp4")
    imageio.mimwrite(video_path, render_images, fps=36)
```

**Parameters:**
- `fps` for video: 36 [S1, unofficial].
- `resolution`: Performance metrics provided for 800x800 and 1352x1014 [S1, unofficial].

**Edge Cases:**
- If `name` is not 'train' or 'test', ground truth is not collected [S1, unofficial].
- If `cam_type` is 'PanopticSports', the GT extraction logic differs, requiring access to `view['image']` instead of `view.original_image` [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of views $N$.
- **Measured Performance**: 82 FPS at 800x800 resolution for synthetic data; 30 FPS at 1352x1014 resolution for real datasets [S1, unofficial].
- **Strengths**: Provides standardized evaluation metrics (SSIM, PSNR, LPIPS) and visualization outputs for 4DGS models.
- **Limitations**: Performance is resolution-dependent and varies between synthetic and real-world datasets.

## Background
The 4DGS Rendering Pipeline originates from the 4D Gaussian Splatting method for dynamic scene reconstruction. It is part of the evaluation and visualization suite provided in the official/unofficial implementation repository [S1, unofficial].

## Sources
[S1] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
