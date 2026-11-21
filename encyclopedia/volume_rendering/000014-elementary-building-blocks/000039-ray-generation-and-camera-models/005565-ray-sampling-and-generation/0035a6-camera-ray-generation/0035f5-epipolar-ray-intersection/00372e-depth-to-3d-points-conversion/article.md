# Depth to 3D Points Conversion

Also called: 3D Point Cloud Generation from Depth

## Overview
This method converts a 2D depth map and associated pseudo-focal lengths into a 3D point cloud in absolute coordinates [S1, unofficial]. It solves the problem of mapping per-pixel depth values from a 2D image plane into 3D space, enabling the construction of geometric scene representations from RGB-D data [S1, unofficial][S2]. Unlike sibling methods that may rely on explicit mesh extraction or virtual laser scans, this approach directly unprojects per-pixel depth values into 3D space, preserving high-dimensional semantic information alongside geometry [S1, unofficial]. It builds on [Epipolar Ray Intersection](7644-epipolar-ray-intersection.md) by providing the specific reprojection logic required to convert rendered buffers into 3D points.

## How it works
**Inputs:**
- `depth`: Depth map of shape $(B, H, W)$ or $(B, H, W, n)$ [S1, unofficial].
- `pseudo_focal`: Pseudo focal lengths of shape $[B, H, W]$, $[B, 2, H, W]$, or $[B, 1, H, W]$ [S1, unofficial].
- `pp` (optional): Principal point coordinates. If `None`, defaults are used [S1, unofficial].

**Outputs:**
- A 3D point cloud represented as coordinates $(x, y, z)$ for each pixel in the depth map [S1, unofficial].

**Data Structures:**
- Grid coordinates `grid_x`, `grid_y`: 2D arrays of shape $(H, W)$ representing the pixel indices [S1, unofficial].
- `pseudo_focalx`, `pseudo_focaly`: Extracted focal length components from `pseudo_focal` [S1, unofficial].

**Steps:**
1. Generate grid coordinates `grid_x` and `grid_y` using an `xy_grid` function, which creates coordinate matrices for the image dimensions $H \times W$ [S1, unofficial].
2. Adjust grid coordinates by subtracting the principal point. If `pp` is provided, subtract `pp`; otherwise, subtract $(W-1)/2$ for x and $(H-1)/2$ for y [S1, unofficial].
3. Extract `pseudo_focalx` and `pseudo_focaly` from `pseudo_focal` based on its shape [S1, unofficial].
4. Compute the x coordinate: $x = \text{depth} \times \text{grid\_x} / \text{pseudo\_focalx}$ [S1, unofficial].
5. Compute the y coordinate: $y = \text{depth} \times \text{grid\_y} / \text{pseudo\_focaly}$ [S1, unofficial].
6. Compute the z coordinate: $z = \text{depth}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def depth_to_3d_points(depth, pseudo_focal, pp=None):
    """
    Converts a depth map and pseudo-focal lengths to a 3D point cloud.
    
    Args:
        depth: np.ndarray of shape (B, H, W) or (B, H, W, n)
        pseudo_focal: np.ndarray of shape [B, H, W], [B, 2, H, W], or [B, 1, H, W]
        pp: tuple (px, py) for principal point. If None, uses image center.
        
    Returns:
        points: np.ndarray of shape (B, H, W, 3) containing x, y, z coordinates
    """
    # Get dimensions
    B, H, W = depth.shape[:3]
    
    # Generate grid coordinates
    # Create meshgrid for x and y indices
    y_indices, x_indices = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
    
    # Adjust grid coordinates by subtracting principal point
    if pp is None:
        px = (W - 1) / 2.0
        py = (H - 1) / 2.0
    else:
        px, py = pp
        
    grid_x = x_indices - px
    grid_y = y_indices - py
    
    # Extract pseudo focal lengths
    # Handle different shapes of pseudo_focal
    if pseudo_focal.ndim == 4:
        if pseudo_focal.shape[1] == 2:
            pseudo_focalx = pseudo_focal[:, 0, :, :]
            pseudo_focaly = pseudo_focal[:, 1, :, :]
        else:
            # Assume single channel or broadcastable
            pseudo_focalx = pseudo_focal[:, 0, :, :]
            pseudo_focaly = pseudo_focal[:, 0, :, :]
    else:
        # Shape [B, H, W]
        pseudo_focalx = pseudo_focal
        pseudo_focaly = pseudo_focal
        
    # Compute coordinates
    # x = depth * grid_x / pseudo_focalx
    # y = depth * grid_y / pseudo_focaly
    # z = depth
    
    # Ensure broadcasting works correctly
    # depth shape: (B, H, W)
    # grid_x, grid_y shape: (H, W) -> broadcast to (B, H, W)
    # pseudo_focalx, pseudo_focaly shape: (B, H, W)
    
    x = depth * grid_x / pseudo_focalx
    y = depth * grid_y / pseudo_focaly
    z = depth
    
    # Stack into (B, H, W, 3)
    points = np.stack([x, y, z], axis=-1)
    
    return points
```

**Parameters:**
- `depth`: Depth map of shape $(B, H, W)$ or $(B, H, W, n)$ [S1, unofficial].
- `pseudo_focal`: Pseudo focal lengths of shape $[B, H, W]$, $[B, 2, H, W]$, or $[B, 1, H, W]$ [S1, unofficial].
- `pp`: Principal point. If `None`, defaults to image center $(W-1)/2, (H-1)/2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero if `pseudo_focalx` or `pseudo_focaly` is zero. [G] Guard against zero focal lengths.
- Shape mismatches between `depth` and `pseudo_focal`. [G] Ensure broadcasting compatibility.
- Principal point offset if not centered. [S1, unofficial] Use provided `pp` or default to center.

## Subtopics

### [Depth Map to Absolute Camera Coordinates](7984-depth-map-to-absolute-camera-coordinates.md)
This method converts 2D rasterized depth and color images into a 3D point cloud using camera intrinsics and extrinsics. It solves the problem of mapping per-pixel depth values from a 2D image plane into 3D world coordinates, enabling the construction of geometric scene representations from multi-view RGB-D data. Unlike sibling methods that may rely on explicit mesh extraction or virtual laser scans, this approach directly unprojects per-pixel depth and learned feature vectors into 3D space, preserving high-dimensional semantic information alongside geometry. It builds on [Depth to 3D Points Conversion](7929-depth-to-3d-points-conversion.md) by providing the specific reprojection logic required to convert rendered buffers into 3D points.

### [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md)
An offline process that generates and refines a 3D point cloud from a subset of training images to improve depth estimation accuracy for novel views in neural rendering. It differs from standard Multi-View Stereo (MVS) by iteratively validating points against ground-truth depth from adjacent views, allowing it to generate a denser, more accurate point cloud for similar computational time, which leads to higher rendering quality (PSNR).

### [Global Scene Point Cloud Construction](8454-global-scene-point-cloud-construction.md)
Global Scene Point Cloud Construction aggregates depth maps from multiple calibrated views into a single, unified 3D point cloud in world coordinates. It solves the problem of redundant and inconsistent primitives that arise when unprojecting overlapping depth maps independently. It differs from [Depth to 3D Points Conversion](7929-depth-to-3d-points-conversion.md) by handling the accumulation and cleaning of multiple views rather than processing a single depth map, and from [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md) by focusing specifically on the initial global assembly and noise removal steps.

### [Fixed-size initialization from RGB-D backprojection](10155-fixed-size-initialization-from-rgb-d-backprojection.md)
This method generates a fixed-size, colored 3D point cloud initialization for volume rendering primitives by backprojecting RGB-D data. It differs from its parent [Depth to 3D Points Conversion](7929-depth-to-3d-points-conversion.md) by applying explicit downsampling factors and stride ratios to control point density, and from its sibling [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md) by relying on direct geometric backprojection rather than implicit field refinement.

## Operating characteristics
- Time complexity: $O(B \cdot H \cdot W)$ for processing all pixels in the batch [S1, unofficial].
- Memory complexity: $O(B \cdot H \cdot W)$ to store the output point cloud [S1, unofficial].
- Measured performance: Not given in the sources.
- Scaling: Linear with respect to image resolution and batch size [S1, unofficial].
- Strengths: Simple, direct conversion from 2D depth to 3D points [S1, unofficial].
- Limitations: Relies on accurate depth maps and pseudo-focal lengths [S1, unofficial].
- When to use: When converting single-view depth maps to 3D points [S1, unofficial].
- When not: When multi-view consistency or refinement is required [S2].

## Background
The method originates from the InstantSplat project, which uses depth maps and pseudo-focal lengths to generate 3D point clouds for neural rendering [S1, unofficial]. It is also referenced in the context of surface-based sampling for improving NeRF models, where depth images from the training set are used to generate point clouds for inference [S2].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
