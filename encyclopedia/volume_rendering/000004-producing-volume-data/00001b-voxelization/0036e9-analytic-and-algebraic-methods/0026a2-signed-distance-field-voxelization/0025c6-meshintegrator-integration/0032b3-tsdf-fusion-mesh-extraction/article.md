# TSDF Fusion Mesh Extraction

Also called: Bounded TSDF Mesh Extraction, ScalableTSDFVolume Mesh Extraction, TSDF Volume Integration

## Overview
This method extracts a triangle mesh from a Truncated Signed Distance Function (TSDF) volume by fusing depth and color observations from multiple viewpoints. It differs from implicit surface extraction methods by relying on explicit depth estimation followed by volumetric integration, offering a balance between geometric fidelity and computational efficiency [S1, unofficial]. It builds on [MeshIntegrator Integration](3630-meshintegrator-integration.md) by consuming the integrated TSDF data to produce a final polygonal representation.

## How it works
**Inputs:**
- A stack of RGB and depth maps from known camera viewpoints.
- Camera intrinsic and extrinsic matrices for each viewpoint.
- Optional binary masks for background removal.
- Parameters: `voxel_size`, `sdf_trunc`, `depth_trunc`.

**Outputs:**
- A triangle mesh (e.g., PLY format).

**Data Structures:**
- `ScalableTSDFVolume`: An Open3D data structure managing the voxel grid. It stores truncated signed distances and integration weights.
- `RGBDImage`: An Open3D structure combining an RGB image and a depth image.

**Parameters:**
- `voxel_size`: Resolution of the TSDF volume. Default: 0.004. [S1, unofficial]
- `sdf_trunc`: Truncation distance for the SDF. Default: 0.02. [S1, unofficial]
- `depth_trunc`: Maximum depth range for integration. Default: 3.0. [S1, unofficial]
- `mask_background`: Whether to mask background pixels. Default: True. [S1, unofficial]

**Steps:**

1. **Initialize Volume:** Create a `ScalableTSDFVolume` instance. Configure it with `voxel_length` set to `voxel_size`, `sdf_trunc` set to `sdf_trunc`, and `color_type` set to `RGB8`. [S1, unofficial]

2. **Iterate Viewpoints:** For each camera in the viewpoint stack:
    a. **Masking:** If `mask_background` is True and a ground truth alpha mask exists, set depth values to 0 where the alpha value is less than 0.5. [S1, unofficial]
    b. **RGBD Creation:** Create an `RGBDImage` from the RGB and depth maps.
        - Clip RGB values to $[0, 1]$.
        - Multiply by 255 and cast to `uint8`.
        - Set depth scale to 1.0.
        - Pass `depth_trunc` to the creator to limit valid depth values. [S1, unofficial]
    c. **Integration:** Integrate the `RGBDImage` into the `ScalableTSDFVolume` using the camera's intrinsic and extrinsic matrices. [S1, unofficial]

3. **Extraction:** Extract the triangle mesh from the integrated volume. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import open3d as o3d
import numpy as np

def extract_mesh_bounded(viewpoint_stack, voxel_size=0.004, sdf_trunc=0.02, depth_trunc=3.0, mask_background=True):
    """
    Extracts a mesh from TSDF volume integrated from depth maps.
    """
    # Step 1: Initialize ScalableTSDFVolume
    # voxel_length: size of one voxel in meters
    # sdf_trunc: truncation distance
    # color_type: RGB8 for color information
    tsdf_vol = o3d.pipelines.integration.ScalableTSDFVolume(
        voxel_length=voxel_size,
        sdf_trunc=sdf_trunc,
        color_type=o3d.pipelines.integration.TSDFVolumeColorType.RGB8
    )

    for viewpoint in viewpoint_stack:
        rgb_map = viewpoint.rgb
        depth_map = viewpoint.depth
        intrinsic = viewpoint.intrinsic
        extrinsic = viewpoint.extrinsic
        alpha_mask = viewpoint.alpha_mask if hasattr(viewpoint, 'alpha_mask') else None

        # Step 2a: Masking
        if mask_background and alpha_mask is not None:
            # Set depth to 0 where alpha < 0.5
            # Note: This modifies the depth map used for integration
            depth_map[alpha_mask < 0.5] = 0.0

        # Step 2b: Create RGBDImage
        # Clip RGB to [0, 1], multiply by 255, cast to uint8
        rgb_uint8 = (np.clip(rgb_map, 0.0, 1.0) * 255).astype(np.uint8)
        
        # Create RGBDImage
        # depth_scale is 1.0 as per source
        rgbd_image = o3d.geometry.RGBDImage.create_from_color_and_depth(
            o3d.geometry.Image(rgb_uint8),
            o3d.geometry.Image(depth_map),
            depth_scale=1.0,
            depth_trunc=depth_trunc,
            convert_rgb_to_intensity=False
        )

        # Step 2c: Integrate
        tsdf_vol.integrate(
            rgbd_image,
            intrinsic=intrinsic,
            extrinsic=extrinsic
        )

    # Step 3: Extract Mesh
    mesh = tsdf_vol.extract_triangle_mesh()
    return mesh
```

**Edge Cases and Pitfalls:**
- **Depth Truncation:** `depth_trunc` must be large enough to cover the scene but small enough to avoid noise from distant objects. It should depend on the scene's scales. [S1, unofficial]
- **Voxel Size:** A smaller `voxel_size` increases resolution but requires more memory and computation. [S1, unofficial]
- **SDF Truncation:** `sdf_trunc` determines the width of the truncation band. Too small a value may lead to noisy surfaces; too large may cause over-smoothing. [S1, unofficial]
- **Background Masking:** If `mask_background` is enabled but no alpha mask is provided, the method may fail or produce incorrect results. [S1, unofficial]

## Subtopics

### [TSDF Reconstruction from Depth Maps](9982-tsdf-reconstruction-from-depth-maps.md)
Converts per-surfel depth outputs from 2D Gaussian Splatting into a unified 3D mesh via TSDF fusion. Chose this when explicit depth estimation is available and volumetric integration is preferred for robustness.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of viewpoints and the number of voxels in the volume. Each integration step processes all pixels in the depth map.
- **Memory Complexity:** Proportional to the volume of the TSDF grid ($N_x \times N_y \times N_z$), where dimensions are determined by `depth_trunc` and `voxel_size`.
- **Strengths:** Robust to noise through volumetric integration; handles multi-view consistency well; produces watertight meshes.
- **Limitations:** Memory intensive for large scenes or high resolutions; requires accurate depth maps and camera poses; truncation parameters need careful tuning.
- **When to Use:** When high-quality mesh extraction from multi-view depth data is required, especially in the context of 2D Gaussian Splatting reconstruction.
- **When Not to Use:** For very large outdoor scenes where memory is constrained, or when real-time performance is critical.

## Background
This method is part of the 2D Gaussian Splatting pipeline, specifically used for mesh extraction from the rendered depth and color maps. It leverages Open3D's `ScalableTSDFVolume` for efficient volumetric integration. The approach was popularized by the 2D Gaussian Splatting project for generating textured triangle meshes from splatted representations. [S1, unofficial]

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
