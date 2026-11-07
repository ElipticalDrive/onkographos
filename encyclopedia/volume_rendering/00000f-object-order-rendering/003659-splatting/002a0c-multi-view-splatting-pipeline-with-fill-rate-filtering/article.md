# Multi-View Splatting Pipeline with Fill-Rate Filtering

Also called: Multi-View Splatting Pipeline

## Overview
This pipeline generates multi-view 2D images from a 3D point cloud by projecting points as splats, filtering out sparse views based on fill rate, and saving structured outputs [S1, unofficial]. It differs from standard ray marching or volume rendering pipelines by operating directly on discrete point primitives rather than continuous density fields, using a screen-space splatting approach that prioritizes geometric coverage validation over volumetric integration [S1, unofficial].

## How it works
**Inputs:**
- Point cloud data: Tuple $(xyz, rgb, label)$ loaded from a `.pt` file, where $xyz \in \mathbb{R}^{N \times 3}$, $rgb \in \mathbb{R}^{N \times 3}$, and $label \in \mathbb{R}^{N}$.
- Camera configuration: Trajectory generation parameters and device specification.
- Rendering parameters: Splat radius $r$, minimum fill rate threshold $\tau$, viewport width $W$, and height $H$.

**Outputs:**
- For each kept view: Intrinsic matrix, extrinsic pose, color image (JPEG), label image (16-bit PNG), color-mapped label (PNG), and index map (`.npy`).

**Data Structures:**
- `CameraPose`: Dataclass containing:
  - `extrinsics`: Tensor $[4, 4]$ representing the world-to-camera transformation matrix.
  - `intrinsics`: Tensor $[4]$ containing camera intrinsic parameters.
  - `position`: Tuple of coordinates.
  - `target`: Tuple of coordinates.
  - `pitch_deg`: Float representing pitch angle.

**Algorithm Steps:**

1.  **Data Loading:** Load the point cloud tuple $(xyz, rgb, label)$ using `torch.load`.
2.  **Trajectory Generation:** Generate camera poses using `generate_trajectory(xyz, config, device)`.
3.  **Batch Preparation:** Extract intrinsics and extrinsics from poses into batches using `poses_to_batch`.
4.  **Renderer Initialization:** Initialize `SplattingRenderer` with the camera batch, splat radius $r$, and device.
5.  **Point Cloud Loading:** Load the point cloud $(xyz, rgb, label)$ into the renderer.
6.  **Rendering:** Render all views by calling `renderer.render_all(intr_batch, extr_batch, W, H)`. This returns lists of color, label, and index maps.
7.  **Fill Rate Calculation:** For each view $i$, compute the fill rate $f_i$ as the mean of valid pixels:
    $$ f_i = \frac{1}{W \times H} \sum_{p \in \text{pixels}} \mathbb{I}(index_i[p] \ge 0) $$
    where $index_i$ is the index map for view $i$, and $\mathbb{I}$ is the indicator function.
8.  **Filtering:** Identify kept indices $K$ where $f_i \ge \tau$.
9.  **Saving:**
    - Save intrinsics from the first kept view to `intrinsic.txt`.
    - For each $i \in K$, save the pose, color (JPEG), label (16-bit PNG), color-mapped label (PNG), and index map (`.npy`).
    - If $K$ is empty, print a warning and skip saving.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from typing import List, Tuple

class SplattingRenderer:
    def __init__(self, cam_batch, splat_radius, device):
        self.cam_batch = cam_batch
        self.splat_radius = splat_radius
        self.device = device
        self.xyz = None
        self.rgb = None
        self.label = None

    def load_pointcloud(self, xyz, rgb, label):
        self.xyz = xyz.to(self.device)
        self.rgb = rgb.to(self.device)
        self.label = label.to(self.device)

    def _project_batch(self, intrinsics, extrinsics, W, H):
        """
        Projects points to screen space.
        Returns:
            uv: [N, 2, M]
            depths: [N, 1, M]
            visible: [N, 1, M]
        """
        # Implementation details for projection are assumed to be handled by the renderer's internal CUDA kernels
        # based on the source code structure provided in [S1, unofficial].
        # This function handles the transformation from 3D world coordinates to 2D screen coordinates.
        pass

    def _assemble_view(self, uv, depths, visible, W, H, point_ids):
        """
        Assembles the final view from projected points, handling depth sorting and splatting.
        """
        N = uv.shape[0]
        Mb = uv.shape[2]
        
        # Flatten for sorting
        flat_uv = uv.reshape(N, 2, -1)
        flat_depths = depths.reshape(N, 1, -1)
        flat_visible = visible.reshape(N, 1, -1)
        
        # Create pixel indices
        pixel_indices = torch.arange(W * H, device=self.device).unsqueeze(0).unsqueeze(0).expand(N, 1, -1)
        
        # Expand point data to match pixel grid for splatting
        # Note: The exact splatting expansion logic involves expanding points to cover a radius r
        # and assigning them to pixels. The source code uses a sort-unique method for depth resolution.
        
        # Simplified representation of the core logic from [S1, unofficial]:
        # 1. Expand points to cover splat radius.
        # 2. Assign to pixels.
        # 3. Sort by pixel index and depth.
        # 4. Keep first entry per pixel (min-depth).
        
        # Placeholder for the complex CUDA-based expansion and sorting logic
        # which is not fully detailed in the text but implied by the source structure.
        # The key is that 'winner_src' and 'winner_flat' are determined by this process.
        
        # Initialize canvases
        color_img = np.zeros((H, W, 3), dtype=np.uint8)
        label_img = np.zeros((H, W), dtype=np.uint16)
        index_map = np.full((H, W), -1, dtype=np.int32)
        
        # The actual filling of these arrays depends on the specific splatting kernel implementation
        # which maps 'winner_src' indices to pixel locations 'winner_flat'.
        # This step is marked as [G] for the specific kernel implementation details not fully explicit in text.
        
        return color_img, label_img, index_map

    def render_all(self, intr_batch, extr_batch, W, H):
        M = intr_batch.shape[0]
        color_list = []
        label_list = []
        index_list = []
        point_ids = torch.arange(self.xyz.shape[0], device=self.device, dtype=torch.int32)

        for start in range(0, M, self.cam_batch):
            end = min(start + self.cam_batch, M)
            sub_intr = intr_batch[start:end].to(self.device)
            sub_extr = extr_batch[start:end].to(self.device)

            uv, depths, visible = self._project_batch(sub_intr, sub_extr, W, H)

            Mb = end - start
            for j in range(Mb):
                color_img, label_img, index_map = self._assemble_view(
                    uv[:, :, j],
                    depths[:, :, j],
                    visible[:, :, j],
                    W,
                    H,
                    point_ids,
                )
                color_list.append(color_img)
                label_list.append(label_img)
                index_list.append(index_map)

        return color_list, label_list, index_list

def main():
    # 1. Load point cloud
    data = torch.load("point_cloud.pt")
    xyz, rgb, label = data
    
    # 2. Generate camera poses
    poses = generate_trajectory(xyz, config, device=torch.device("cpu"))
    
    # 3. Extract intrinsics and extrinsics
    intr_batch, extr_batch = poses_to_batch(poses)
    
    # 4. Initialize Renderer
    renderer = SplattingRenderer(cam_batch=16, splat_radius=2, device=torch.device("cuda"))
    renderer.load_pointcloud(xyz, rgb, label)
    
    # 5. Render all views
    color_list, label_list, index_list = renderer.render_all(intr_batch, extr_batch, 1024, 1024)
    
    # 6. Compute fill rates
    fill_rates = []
    for index_map in index_list:
        # Count valid pixels (index >= 0)
        valid_pixels = np.count_nonzero(index_map >= 0)
        total_pixels = index_map.shape[0] * index_map.shape[1]
        fill_rates.append(valid_pixels / total_pixels)
        
    # 7. Filter views
    min_fill_rate = 0.30
    kept_indices = [i for i, f in enumerate(fill_rates) if f >= min_fill_rate]
    
    # 8. Save outputs
    if not kept_indices:
        print("Warning: No views meet the fill rate threshold.")
        return
        
    # Save intrinsics from first kept view
    # Assuming intr_batch is accessible or stored in renderer
    # intrinsic.txt saving logic here
    
    for i in kept_indices:
        # Save pose, color (JPEG), label (16-bit PNG), color-mapped label (PNG), index map (.npy)
        pass

if __name__ == "__main__":
    main()
```

**Parameters:**
- `splat_radius`: Pixel expansion radius for denser coverage. Default value is 2, which corresponds to a $5 \times 5$ disk [S1, unofficial].
- `min_fill_rate`: Minimum fraction of valid pixels ($index \ge 0$) to keep a view. Default value is 0.30 [S1, unofficial].

**Edge Cases and Pitfalls:**
- If all views fall below `min_fill_rate`, no output is saved, and a warning is printed [S1, unofficial].
- Precision issues may arise in depth sorting if not handled by the guaranteed correct CUDA sort-unique method described in the source [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the projection and splatting operations, which are $O(N \cdot M)$ where $N$ is the number of points and $M$ is the number of views. The fill rate calculation is $O(M \cdot W \cdot H)$.
- **Memory Complexity:** Requires storage for the point cloud and intermediate buffers for each view. The batched rendering helps manage GPU memory usage.
- **Strengths:** Efficient generation of multi-view data from point clouds; automatic filtering of poor-quality views ensures dataset quality.
- **Limitations:** Relies on point cloud data; splatting may introduce artifacts if the radius is not chosen appropriately; fill rate filtering may discard useful views if the threshold is too high.
- **When to use:** Suitable for generating training data for multi-view learning tasks where high-quality, dense views are required.
- **When not to use:** Not suitable for real-time rendering or when continuous volumetric information is needed.

## Background
This pipeline is used in the DALA-CitywallV3 dataset pipeline for generating multi-view splatting data [S1, unofficial]. It represents a practical approach to preparing multi-view datasets from 3D point cloud scans, leveraging GPU acceleration for efficient rendering.

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
