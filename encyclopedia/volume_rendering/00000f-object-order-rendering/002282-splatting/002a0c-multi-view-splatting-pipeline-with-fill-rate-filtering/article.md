# Multi-View Splatting Pipeline with Fill-Rate Filtering

Also called: Multi-View Splatting Pipeline

## Overview
A rendering pipeline that generates multi-view 2D images from a 3D point cloud, filters out sparse views based on fill rate, and saves structured outputs [S1, unofficial]. It differs from standard splatting by explicitly discarding camera poses that do not provide sufficient pixel coverage, ensuring high-quality data for downstream tasks like dataset generation [S1, unofficial].

## How it works
**Inputs:**
- Point cloud data: tuple `(xyz, rgb, label)` loaded from a `.pt` file [S1, unofficial].
- Configuration: `splat_radius` (default 2), `min_fill_rate` (default 0.30) [S1, unofficial].
- Target resolution: `width` ($W$), `height` ($H$) [S1, unofficial].

**Outputs:**
- `intrinsic.txt`: Camera intrinsics from the first kept view [S1, unofficial].
- For each kept view:
  - Pose file (extrinsics) [S1, unofficial].
  - Color image (JPEG) [S1, unofficial].
  - Label image (16-bit PNG) [S1, unofficial].
  - Color-mapped label image (PNG) [S1, unofficial].
  - Index map (`.npy`) [S1, unofficial].

**Data Structures:**
- `CameraPose`:
  - `extrinsics`: `torch.Tensor` of shape `[4, 4]` [S1, unofficial].
  - `intrinsics`: `torch.Tensor` of shape `[4]` [S1, unofficial].
  - `position`: `tuple` [S1, unofficial].
  - `target`: `tuple` [S1, unofficial].
  - `pitch_deg`: `float` [S1, unofficial].

**Algorithm:**
1. **Load Data:** Load point cloud `(xyz, rgb, label)` using `torch.load` [S1, unofficial].
2. **Generate Poses:** Compute camera poses using `generate_trajectory(xyz, config, device="cpu")` [S1, unofficial].
3. **Batch Poses:** Extract intrinsics and extrinsics into batches via `poses_to_batch(poses)` [S1, unofficial].
4. **Initialize Renderer:** Create `SplattingRenderer` with `cam_batch`, `splat_radius`, and `device` [S1, unofficial].
5. **Load Point Cloud:** Pass `(xyz, rgb, label)` to `renderer.load_pointcloud()` [S1, unofficial].
6. **Render Views:** Call `renderer.render_all(intr_batch, extr_batch, width, height)` to get `color_list`, `label_list`, and `index_list` [S1, unofficial].
7. **Compute Fill Rates:** For each view $i$, calculate fill rate $f_i$:
   $$ f_i = \frac{\text{count}(index\_list[i] \ge 0)}{W \times H} $$
   where $index\_list[i]$ is the index map for view $i$ [S1, unofficial].
8. **Filter Views:** Identify kept indices $K$:
   $$ K = \{ i \mid f_i \ge \text{min\_fill\_rate} \} $$
   [S1, unofficial].
9. **Save Outputs:**
   - If $K$ is empty, print a warning and skip saving [S1, unofficial].
   - Save intrinsics from view $K[0]$ to `intrinsic.txt` [S1, unofficial].
   - For each $i \in K$, save pose, color (JPEG), label (16-bit PNG), color-mapped label (PNG), and index map (`.npy`) [S1, unofficial].

**Splatting Implementation Details:**
The renderer uses a CUDA-accelerated approach. For each camera, it projects points to screen space. To handle depth sorting and avoid artifacts, it uses a sort-unique strategy:
1. Expand points into a flat list of potential pixel contributions.
2. Compute a sort key: $sort\_key = flat\_exp \times (d_{src}.max() + 1.0) + d_{src}$, where $flat\_exp$ encodes pixel index and $d_{src}$ is depth [S1, unofficial].
3. Sort by this key.
4. Keep the first entry for each unique pixel index (guaranteeing minimum depth) [S1, unofficial].
5. Write canvases using the winner indices [S1, unofficial].

**EWA Splatting Variant:**
An optional EWA (Elliptical Weighted Average) mode uses k-nearest neighbors to estimate 3D covariance, then projects it to 2D for denser coverage [S1, unofficial]. This is more expensive and intended for offline rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from typing import List, Tuple

def multi_view_splatting_pipeline(
    point_cloud_path: str,
    config: dict,
    width: int,
    height: int,
    splat_radius: int = 2,
    min_fill_rate: float = 0.30,
    device: str = "cuda"
) -> None:
    # 1. Load point cloud
    xyz, rgb, label = torch.load(point_cloud_path)
    
    # 2. Generate camera poses
    poses = generate_trajectory(xyz, config, device=torch.device("cpu"))
    
    # 3. Extract intrinsics and extrinsics
    intr_batch, extr_batch = poses_to_batch(poses)
    
    # 4. Initialize renderer
    renderer = SplattingRenderer(
        cam_batch=16, # Example batch size
        splat_radius=splat_radius,
        device=device
    )
    renderer.load_pointcloud(xyz, rgb, label)
    
    # 5. Render all views
    color_list, label_list, index_list = renderer.render_all(
        intr_batch, extr_batch, width, height
    )
    
    # 6. Compute fill rates
    fill_rates = []
    for idx_map in index_list:
        # idx_map is numpy array [H, W]
        valid_pixels = np.count_nonzero(idx_map >= 0)
        total_pixels = width * height
        fill_rate = valid_pixels / total_pixels
        fill_rates.append(fill_rate)
    
    # 7. Filter views
    kept_indices = [i for i, f in enumerate(fill_rates) if f >= min_fill_rate]
    
    if not kept_indices:
        print("Warning: No views meet the minimum fill rate threshold.")
        return
    
    # 8. Save outputs
    # Save intrinsics from first kept view
    first_kept = kept_indices[0]
    intrinsics = intr_batch[first_kept]
    save_intrinsics(intrinsics, "intrinsic.txt")
    
    for i in kept_indices:
        save_pose(poses[i], f"pose_{i}.txt")
        save_image(color_list[i], f"color_{i}.jpg", format="JPEG")
        save_image(label_list[i], f"label_{i}.png", format="PNG16")
        save_image(color_mapped_label(label_list[i]), f"label_mapped_{i}.png", format="PNG")
        save_array(index_list[i], f"index_{i}.npy")

def save_intrinsics(intrinsics, filename):
    np.savetxt(filename, intrinsics.cpu().numpy())

def save_pose(pose, filename):
    # Assuming pose is a CameraPose object or dict
    np.savetxt(filename, pose.extrinsics.cpu().numpy())

def save_image(img, filename, format="JPEG"):
    # img is numpy array [H, W, 3] or [H, W]
    if format == "JPEG":
        # Convert to uint8 if needed
        if img.dtype != np.uint8:
            img = (img * 255).astype(np.uint8)
        from PIL import Image
        Image.fromarray(img).save(filename, format="JPEG")
    elif format == "PNG16":
        if img.dtype != np.uint16:
            img = img.astype(np.uint16)
        from PIL import Image
        Image.fromarray(img).save(filename, format="PNG")

def save_array(arr, filename):
    np.save(filename, arr)

def color_mapped_label(label_img):
    # Placeholder for color mapping logic
    return label_img
```

## Operating characteristics
- **Time Complexity:** Dominated by point projection and sorting. EWA mode adds kNN search and covariance computation, increasing cost [S1, unofficial].
- **Memory Complexity:** Scales with point cloud size and number of views. Sub-batching is used to fit GPU memory [S1, unofficial].
- **Strengths:** Ensures high-quality multi-view data by filtering out sparse views; CUDA-accelerated for speed [S1, unofficial].
- **Limitations:** EWA mode is slow; requires sufficient point density for valid fill rates [S1, unofficial].
- **When to Use:** Dataset generation for multi-view learning tasks (e.g., DALA-CitywallV3) [S1, unofficial].
- **When Not to Use:** Real-time rendering or scenes with very sparse point clouds [S1, unofficial].

## Background
Originated in the context of generating multi-view splatting data for 3D reconstruction and novel view synthesis. Used in the DALA-CitywallV3 dataset pipeline [S1, unofficial].

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
