# Multi-Plane Image (MPI) Rendering from Point Clouds

Also called: MPI Rendering, Multi-Plane Projection

## Overview
Multi-Plane Image (MPI) Rendering from Point Clouds is a preprocessing technique that renders a 3D point cloud into a set of 2D image planes (MPI) by projecting points onto a target image grid and handling occlusion between planes [S1, unofficial]. It aggregates points into discrete depth bins and blends them front-to-back using a visibility mask [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point cloud dataset loaded from a PLY file, containing positions stored as $[x, y, z, 1.0]$ and colors stored as $[r, g, b]$ [S1, unofficial].
*   **Parameters:** `num_planes` (number of depth planes, typically 32) and `target_h`, `target_w` (output image height and width, typically 480 and 640 for ScanNet) [S1, unofficial].
*   **Output:** A final rendered image tensor of shape $[H, W, 3]$ and a final mask tensor of shape $[H, W, 1]$ [S1, unofficial].

**Data Structures:**
1.  **Intermediate Tensors:** `image_all` of shape $[num\_planes, H, W, 3]$ and `mask_all` of shape $[num\_planes, H, W, 1]$, both initialized to zeros [S1, unofficial]. `image_all` stores the color contribution of points assigned to each plane, while `mask_all` stores the validity mask for each plane [S1, unofficial].
2.  **Final Tensors:** `final_image` of shape $[H, W, 3]$ and `final_mask` of shape $[H, W, 1]$, both initialized to zeros [S1, unofficial].

**Steps:**
1.  Initialize `image_all` and `mask_all` tensors of shape $[num\_planes, H, W, 3]$ and $[num\_planes, H, W, 1]$ respectively, filled with zeros [S1, unofficial].
2.  For each point in the point cloud, determine its assigned plane index $d$ based on its depth $z$, project the point's 3D position to 2D image coordinates $(u, v)$, assign the point's color to `image_all[d, v, u, :]`, and set `mask_all[d, v, u, 0] = 1` to mark the position as occupied [S1, unofficial].
3.  Initialize `final_image` and `final_mask` tensors of shape $[H, W, 3]$ and $[H, W, 1]$ with zeros [S1, unofficial].
4.  Iterate through each plane $i$ from $0$ to $num\_planes - 1$:
    *   If $i = 0$, copy `image_all[0]` to `final_image` and `mask_all[0]` to `final_mask` [S1, unofficial].
    *   If $i > 0$, calculate the valid region as the part of the current plane's mask not yet covered by the accumulated mask: $valid = mask\_all[i] \times (1 - final\_mask)$ [S1, unofficial].
    *   Update the final image by blending the new plane's image in the valid region: $final\_image = final\_image \times final\_mask + valid \times image\_all[i]$ [S1, unofficial].
    *   Update the accumulated mask by adding the valid region: $final\_mask = final\_mask + valid$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_mpi(point_clouds, point_clouds_colors, num_planes, target_h, target_w, 
               fx, fy, cx, cy, z_min, z_max):
    """
    Render a point cloud into a Multi-Plane Image.
    
    Args:
        point_clouds: Array of shape [N, 4] containing [x, y, z, 1.0]
        point_clouds_colors: Array of shape [N, 3] containing [r, g, b]
        num_planes: Integer, number of depth planes
        target_h: Integer, output height
        target_w: Integer, output width
        fx, fy, cx, cy: Camera intrinsic parameters
        z_min, z_max: Depth range for plane binning
    Returns:
        final_image: Array of shape [target_h, target_w, 3]
        final_mask: Array of shape [target_h, target_w, 1]
    """
    # Initialize intermediate tensors
    image_all = np.zeros((num_planes, target_h, target_w, 3), dtype=np.float32)
    mask_all = np.zeros((num_planes, target_h, target_w, 1), dtype=np.float32)
    
    # Project points to planes
    for i in range(point_clouds.shape[0]):
        pos = point_clouds[i]
        color = point_clouds_colors[i]
        z = pos[2]
        
        # Standard perspective projection to 2D grid coordinates
        u = int((fx * pos[0] / z) + cx)
        v = int((fy * pos[1] / z) + cy)
        
        # Depth-to-plane mapping
        d = int(np.clip((z - z_min) / (z_max - z_min) * num_planes, 0, num_planes - 1))
        
        # Assign color and mask if within bounds
        if 0 <= u < target_w and 0 <= v < target_h:
            image_all[d, v, u, :] = color
            mask_all[d, v, u, 0] = 1.0

    # Initialize final tensors
    final_image = np.zeros((target_h, target_w, 3), dtype=np.float32)
    final_mask = np.zeros((target_h, target_w, 1), dtype=np.float32)

    # Blend planes front-to-back
    for i in range(num_planes):
        if i == 0:
            final_image = image_all[i].copy()
            final_mask = mask_all[i].copy()
        else:
            # Calculate valid region: current plane's mask minus accumulated mask
            valid = mask_all[i] * (1.0 - final_mask)
            
            # Blend image
            final_image = final_image * final_mask + valid * image_all[i]
            
            # Update mask
            final_mask = final_mask + valid

    return final_image, final_mask
```

**Edge Cases and Pitfalls:**
*   **PLY Parsing:** The source assumes the PLY header ends with the line `end_header`. Lines following this contain position (indices 0-2) and color (indices 3-5) data. Colors are read as BGR (indices 5, 4, 3) and must be converted to RGB [S1, unofficial].
*   **Boundary Checks:** Points projecting outside the $[0, H) \times [0, W)$ grid must be discarded to avoid index errors [G].
*   **Depth Binning:** Points with depth $z$ outside the $[z\_min, z\_max]$ range must be clamped or discarded to prevent incorrect plane assignment [G].

## Subtopics

### Multi-Plane Image and Weight Rendering
A rendering technique that produces multiple planes of images with their blending weights, where each plane corresponds to a specific depth in camera coordinates [S2]. The blending weights roughly reflect the scene depth, and for each pixel, the plane with the correct depth provides the right color [S2]. It is chosen when a neural network outputs explicit blending weights alongside RGB planes. Compared to z-buffer based projection, the camera frustum based point projection used here is more robust when the point cloud is sparse, especially on large scenes like Matterport3D, resulting in more complete renders that reflect correct occlusion [S2].

## Operating characteristics
Not given in the sources.

## Background
The method is described in "Neural Point Cloud Rendering via Multi-Plane Projection" [S2]. Implementation details for point cloud simplification and MPI generation are available in the associated repository [S1, unofficial].

## Sources
[S1] PCMP-NeRF, https://github.com/Mayxmu/PCMP-NeRF, https://github.com/Mayxmu/PCMP_NeRF, fetched 2026-10-30 (unofficial)
[S2] Neural Point Cloud Rendering via Multi-Plane Projection, https://doi.org/10.1109/cvpr42600.2020.00785, fetched 2026-10-30
