# TSDF Volume Integration

Also called: ScalableTSDFVolume Integration, TSDF Truncation and Integration

## Overview
TSDF Volume Integration builds on [MeshIntegrator Integration](3630-meshintegrator-integration.md) to write a triangle mesh's truncated distance field into a `VoxelBlockGrid`'s `tsdf` and `weight` attributes [Parent]. It accumulates truncated signed distance fields from multiple RGB-D views into a voxel grid to reconstruct a 3D surface [S1, unofficial]. It differs from algebraic volume rendering methods by explicitly modeling surface proximity via a truncation function, and from pure mesh extraction techniques by operating in a volumetric space before surface reconstruction [S1, unofficial].

## How it works
**Inputs:**
- A list of camera poses (extrinsics) and intrinsics [S1, unofficial].
- Corresponding color (RGB) and depth images [S1, unofficial].
- Parameters: `voxel_size`, `sdf_trunc`, `max_depth`, `downsample_scale` [S1, unofficial].

**Outputs:**
- A `ScalableTSDFVolume` object containing the integrated `tsdf` and `weight` fields [S1, unofficial].
- A triangle mesh extracted via `extract_triangle_mesh()` [S1, unofficial].

**Data Structures:**
The core data structure is `ScalableTSDFVolume`, which manages a `VoxelBlockGrid` with `tsdf` and `weight` attributes [Parent]. Each active voxel stores a float `tsdf` value clamped to $[-trunc, trunc]$ and a float `weight` value [Parent].

**Parameters:**
- `voxel_size`: Length of a voxel edge in world units. Default: 0.02 [S1, unofficial].
- `sdf_trunc`: Truncation distance. Distances beyond this are clamped. Default: 0.04 [S1, unofficial].
- `max_depth`: Maximum depth value to consider. Depth values $> max\_depth$ are ignored. Default: 5.0 [S1, unofficial].
- `downsample_scale`: Factor to reduce image resolution for faster processing. Default: 1 [S1, unofficial].

**Steps:**
1. Initialize `ScalableTSDFVolume` with `voxel_length`, `sdf_trunc`, and `color_type=RGB8` [S1, unofficial].
2. Iterate through the dataset cameras [S1, unofficial].
3. If `downsample_scale > 1` and the camera has a `downsample_scale` method, downsample the camera [S1, unofficial].
4. Skip frames if depth is unavailable (`cam.depth` is `None`) [S1, unofficial].
5. Convert color to uint8 RGB $(H \times W \times 3)$, handling $C \times H \times W$ transposition and float $[0,1]$ scaling [S1, unofficial].
6. Convert depth to float32, clamp invalid values to 0, and truncate far depths beyond `max_depth` [S1, unofficial].
7. Build camera intrinsics. Prefer explicit $f_x, f_y, c_x, c_y$. Fallback to computing from FoV: $f_x = w / (2.0 \tan(\text{FoV}_x / 2.0))$. Last resort fallback sets $f_x = f_y = 0.5(w + h)$ [S1, unofficial].
8. Build world-to-camera extrinsic matrix $E$ (4x4). Try attributes in order: `extrinsics`, `w2c`, `inv(c2w)`, or construct from $R$ and $T$. Ensure the matrix is 4x4 [S1, unofficial].
9. Create an `RGBDImage` from the processed color and depth data: `RGBDImage.create_from_color_and_depth(color, depth, depth_scale=1.0, depth_trunc=max_depth, convert_rgb_to_intensity=False)` [S1, unofficial].
10. Integrate the `RGBDImage` into the volume: `volume.integrate(rgbd, intrinsic, extrinsic)` [S1, unofficial]. Voxels within the truncation band are overwritten (not blended), clamped to $\pm sdf\_trunc$ with `weight = 1`. Voxels outside the band in those blocks get `tsdf = 0`, `weight = 0`. Blocks outside the band are untouched [Parent].
11. Extract the final triangle mesh: `mesh = volume.extract_triangle_mesh()` and compute vertex normals [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import open3d as o3d

def integrate_tsdf(cameras, color_images, depth_images, voxel_size=0.02, sdf_trunc=0.04, max_depth=5.0, downsample_scale=1):
    volume = o3d.pipelines.integration.ScalableTSDFVolume(
        voxel_length=voxel_size,
        sdf_trunc=sdf_trunc,
        color_type=o3d.pipelines.integration.TSDFVolumeColorType.RGB8
    )

    for i, cam in enumerate(cameras):
        color_img = color_images[i]
        depth_img = depth_images[i]

        if depth_img is None:
            continue

        if downsample_scale > 1 and hasattr(cam, 'downsample_scale'):
            color_img, depth_img = cam.downsample_scale(color_img, depth_img, downsample_scale)

        # Handle CxHxW transposition and float [0,1] scaling if needed [G]
        if color_img.ndim == 3 and color_img.shape[0] == 3:
            color_img = np.transpose(color_img, (1, 2, 0))
        if color_img.dtype == np.float32 or color_img.dtype == np.float64:
            color_img = (color_img * 255).astype(np.uint8)

        depth_float = depth_img.astype(np.float32)
        depth_float[depth_float > max_depth] = 0.0
        depth_float[depth_float <= 0] = 0.0

        w, h = color_img.shape[1], color_img.shape[0]

        if hasattr(cam, 'fx'):
            fx, fy, cx, cy = cam.fx, cam.fy, cam.cx, cam.cy
        elif hasattr(cam, 'fov_x'):
            fx = w / (2.0 * np.tan(cam.fov_x / 2.0))
            fy = h / (2.0 * np.tan(cam.fov_y / 2.0))
            cx, cy = w / 2.0, h / 2.0
        else:
            fx = fy = 0.5 * (w + h)
            cx, cy = w / 2.0, h / 2.0

        intrinsic = o3d.camera.PinholeCameraIntrinsic(
            width=w, height=h, fx=fx, fy=fy, cx=cx, cy=cy
        )

        if hasattr(cam, 'extrinsics'):
            extrinsic = cam.extrinsics
        elif hasattr(cam, 'w2c'):
            extrinsic = cam.w2c
        elif hasattr(cam, 'c2w'):
            extrinsic = np.linalg.inv(cam.c2w)
        else:
            extrinsic = np.eye(4)
            extrinsic[:3, :3] = cam.R
            extrinsic[:3, 3] = cam.T

        rgbd = o3d.geometry.RGBDImage.create_from_color_and_depth(
            o3d.geometry.Image(color_img),
            o3d.geometry.Image(depth_float),
            depth_scale=1.0,
            depth_trunc=max_depth,
            convert_rgb_to_intensity=False
        )

        volume.integrate(rgbd, intrinsic, extrinsic)

    mesh = volume.extract_triangle_mesh()
    mesh.compute_vertex_normals()
    return mesh
```

**Edge Cases:**
- Invalid depth values ($> max\_depth$ or $\le 0$) are clamped to 0 and ignored during integration [S1, unofficial].
- Precision: Depth is converted to float32 for processing [S1, unofficial].

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Performance: Not given in the sources.
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.

## Background
The method originates from the Open3D library's integration pipeline, where the `ScalableTSDFVolume` class provides a memory-efficient way to integrate RGB-D data [S1, unofficial]. It is used in 3D reconstruction pipelines to fuse multi-view depth observations into a coherent volumetric representation [S1, unofficial].

## Sources
[S1] GAP-LAB-CUHK-SZ/gaustudio, https://github.com/GAP-LAB-CUHK-SZ/gaustudio, fetched 2026-10-26 (unofficial)
