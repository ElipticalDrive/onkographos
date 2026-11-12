# Multi-View Splatting Pipeline with Fill-Rate Filtering

Also called: Multi-View Splatting Pipeline

## Overview
This method generates multi-view 2D images from a 3D point cloud by rasterizing points as splats, then filters the resulting views based on pixel coverage density [S1, unofficial]. It distinguishes itself from standard splatting by explicitly discarding views with insufficient fill rate, ensuring structured output quality for downstream tasks like dataset generation [S1, unofficial].

## How it works
**Inputs:**
- Point cloud data: tuple $(\mathbf{xyz}, \mathbf{rgb}, \mathbf{label})$ loaded from a `.pt` file [S1, unofficial].
- Camera configuration: trajectory parameters and intrinsic/extrinsic matrices [S1, unofficial].
- Parameters: `splat_radius` (default 2, covering a $5 \times 5$ disk), `min_fill_rate` (default 0.30), viewport width $W$, height $H$ [S1, unofficial].

**Outputs:**
- For each kept view: camera pose, color image (JPEG), label image (16-bit PNG), color-mapped label (PNG), and index map (`.npy`) [S1, unofficial].
- Intrinsics from the first kept view saved to `intrinsic.txt` [S1, unofficial].

**Data Structures:**
- `CameraPose`: Dataclass containing `extrinsics` (Tensor $[4,4]$), `intrinsics` (Tensor $[4]$), `position` (tuple), `target` (tuple), `pitch_deg` (float) [S1, unofficial].
- `SplattingRenderer`: Holds the point cloud and rendering state [S1, unofficial].

**Steps:**
1. **Load Data:** Load point cloud $(\mathbf{xyz}, \mathbf{rgb}, \mathbf{label})$ using `torch.load` [S1, unofficial].
2. **Generate Poses:** Compute camera poses using `generate_trajectory(xyz, config, device)` [S1, unofficial].
3. **Batch Poses:** Extract intrinsics and extrinsics into batches via `poses_to_batch(poses)` [S1, unofficial].
4. **Initialize Renderer:** Create `SplattingRenderer` with `cam_batch`, `splat_radius`, and device [S1, unofficial].
5. **Load Point Cloud:** Pass $(\mathbf{xyz}, \mathbf{rgb}, \mathbf{label})$ to `renderer.load_pointcloud()` [S1, unofficial].
6. **Render Views:** Call `renderer.render_all(intr_batch, extr_batch, W, H)` to get `color_list`, `label_list`, `index_list` [S1, unofficial].
7. **Compute Fill Rates:** For each view $i$, calculate fill rate $f_i = \frac{1}{H \cdot W} \sum_{x,y} \mathbb{I}(\text{index}_i[x,y] \ge 0)$ [S1, unofficial].
8. **Filter Views:** Keep indices $i$ where $f_i \ge \text{min\_fill\_rate}$ [S1, unofficial].
9. **Save Outputs:** For each kept view, save pose, color, label, mapped label, and index map [S1, unofficial]. If no views meet the threshold, print a warning and skip saving [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from typing import List, Tuple

def multi_view_splatting_pipeline(
    point_cloud_path: str,
    config: dict,
    W: int,
    H: int,
    splat_radius: int = 2,
    min_fill_rate: float = 0.30,
    device: str = "cpu"
) -> None:
    # 1. Load point cloud
    data = torch.load(point_cloud_path)
    xyz, rgb, label = data['xyz'], data['rgb'], data['label']
    
    # 2. Generate camera poses
    poses = generate_trajectory(xyz, config, device=torch.device(device))
    
    # 3. Extract intrinsics and extrinsics
    intr_batch, extr_batch = poses_to_batch(poses)
    
    # 4. Initialize renderer
    renderer = SplattingRenderer(
        cam_batch=1, # Adjust based on memory
        splat_radius=splat_radius,
        device=device
    )
    
    # 5. Load point cloud into renderer
    renderer.load_pointcloud(xyz, rgb, label)
    
    # 6. Render all views
    color_list, label_list, index_list = renderer.render_all(
        intr_batch, extr_batch, W, H
    )
    
    # 7. Compute fill rates
    fill_rates = []
    for idx_map in index_list:
        # idx_map is [H, W] int32; valid pixels have idx >= 0
        valid_pixels = (idx_map >= 0).sum()
        total_pixels = H * W
        fill_rates.append(valid_pixels / total_pixels)
    
    # 8. Filter views
    kept_indices = [i for i, f in enumerate(fill_rates) if f >= min_fill_rate]
    
    if not kept_indices:
        print("Warning: No views meet the minimum fill rate threshold.")
        return
    
    # 9. Save outputs
    # Save intrinsics from the first kept view
    first_kept = kept_indices[0]
    # Assuming intrinsics are accessible via renderer or poses
    # This step depends on specific API access to pose data
    save_intrinsics(intr_batch[first_kept], "intrinsic.txt")
    
    for i in kept_indices:
        save_pose(poses[i], f"pose_{i}.txt")
        save_image(color_list[i], f"color_{i}.jpg", format="JPEG")
        save_image(label_list[i], f"label_{i}.png", format="PNG-16")
        save_image(color_map_label(label_list[i]), f"label_mapped_{i}.png")
        save_array(index_list[i], f"index_{i}.npy")

def save_intrinsics(intrinsics: torch.Tensor, filename: str):
    np.savetxt(filename, intrinsics.cpu().numpy())

def save_pose(pose, filename: str):
    # Implementation depends on pose structure
    pass

def save_image(img: np.ndarray, filename: str, format: str):
    # Implementation depends on image library
    pass

def color_map_label(label: np.ndarray) -> np.ndarray:
    # Map label values to colors
    pass
```

**Edge Cases and Pitfalls:**
- If all views fall below `min_fill_rate`, no output is saved and a warning is printed [S1, unofficial].
- Precision issues in depth sorting may affect which point wins in overlapping splats; the implementation uses a sort-unique strategy on CUDA to ensure deterministic results [S1, unofficial].

## Subtopics

### EWA Splatting
Uses elliptical weighted average splatting with per-point 3D covariance estimated from k-nearest neighbours for denser coverage [S1, unofficial]. More expensive, intended for high-quality offline rendering.

### Standard Splatting
Uses a fixed `splat_radius` for pixel expansion. Faster, suitable for real-time or large-scale batch processing [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the kNN search in EWA mode ($O(N \log N)$ for tree construction, $O(N \cdot k)$ for query) and rasterization ($O(N)$ for splatting, $O(W \cdot H)$ for assembly) [S1, unofficial].
- **Memory Complexity:** $O(N)$ for point cloud storage, $O(W \cdot H)$ for intermediate buffers per view [S1, unofficial].
- **Performance:** EWA is significantly more expensive than standard splatting due to covariance estimation [S1, unofficial].
- **Strengths:** Ensures high-quality, dense coverage in output views by filtering sparse ones [S1, unofficial].
- **Limitations:** Computationally intensive for EWA; may discard useful views if `min_fill_rate` is set too high [S1, unofficial].

## Background
This pipeline is used in the DALA-CitywallV3 dataset generation process [S1, unofficial]. It leverages PyTorch and CUDA for efficient point cloud processing and rendering [S1, unofficial].

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
