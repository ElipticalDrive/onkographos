# TSDF Volume Integration

Also called: ScalableTSDFVolume Integration, TSDF Truncation and Integration

## Overview
TSDF Volume Integration is a method for reconstructing a 3D surface from a set of RGB-D images by accumulating truncated signed distance fields into a voxel grid. It solves the problem of noisy or incomplete depth data by weighting each voxel's distance value by the confidence of the measurement, effectively filtering out outliers and filling gaps. It differs from algebraic methods (like MIP or standard volume rendering) by explicitly modeling surface proximity via a truncation function, and from pure mesh extraction techniques by operating in a volumetric space before marching cubes.

## How it works
**Inputs:**
- A list of camera poses (extrinsics) and intrinsics.
- Corresponding color (RGB) and depth images.
- Parameters: `voxel_size`, `sdf_trunc`, `max_depth`, `downsample_scale`.

**Outputs:**
- A `ScalableTSDFVolume` object containing the integrated TSDF and weight fields.
- A triangle mesh extracted via `extract_triangle_mesh()`.

**Data Structures:**
The core data structure is `ScalableTSDFVolume`, implemented in Open3D. It maintains a sparse voxel grid. Each active voxel stores:
- `tsdf`: Float value, clamped to $[-trunc, trunc]$.
- `weight`: Float value, accumulated confidence.
- `color`: Optional RGB value (if `color_type` is not NONE).

**Parameters:**
- `voxel_size`: Length of a voxel edge in world units. Default: 0.02 [S1, unofficial].
- `sdf_trunc`: Truncation distance. Distances beyond this are clamped. Default: 0.04 [S1, unofficial].
- `max_depth`: Maximum depth value to consider. Depth values $> max\_depth$ are ignored. Default: 5.0 [S1, unofficial].
- `downsample_scale`: Factor to reduce image resolution for faster processing. Default: 1 [S1, unofficial].

**Steps:**

1. **Initialization**: Create the volume.
   $$ V = \text{ScalableTSDFVolume}(\text{voxel\_length}=v, \text{sdf\_trunc}=t, \text{color\_type}=\text{RGB8}) $$
   Where $v$ is `voxel_size` and $t$ is `sdf_trunc` [S1, unofficial].

2. **Camera Preprocessing**:
   - If `downsample_scale > 1`, downsample color and depth images by this factor.
   - Skip frames where depth is unavailable.

3. **Intrinsic Construction**:
   - Prefer explicit $f_x, f_y, c_x, c_y$.
   - If only Field of View (FoV) is available:
     $$ f_x = \frac{w}{2 \tan(\frac{\text{FoV}_x}{2})} $$
     $$ f_y = \frac{h}{2 \tan(\frac{\text{FoV}_y}{2})} $$
   - Fallback if FoV is missing: $f_x = f_y = 0.5(w+h)$ [S1, unofficial].

4. **Extrinsic Construction**:
   - Build $4 \times 4$ world-to-camera matrix $E$.
   - Sources: `extrinsics`, `w2c`, `inv(c2w)`, or construct from Rotation $R$ and Translation $T$ [S1, unofficial].

5. **RGBD Image Creation**:
   - Convert color to uint8 RGB $(H \times W \times 3)$.
   - Convert depth to float32, clamp invalid values to 0, truncate $> max\_depth$.
   - Create `RGBDImage`:
     $$ \text{RGBD} = \text{RGBDImage.create\_from\_color\_and\_depth}(C, D, \text{depth\_scale}=1.0, \text{depth\_trunc}=max\_depth) $$
     [S1, unofficial].

6. **Integration**:
   - For each camera $i$ with pose $E_i$ and intrinsics $K$:
     $$ V.\text{integrate}(\text{RGBD}_i, K, E_i) $$
   - Inside `integrate()`:
     - For each pixel $(u,v)$ in the depth image:
       - Project to 3D point $P$ using $K^{-1}$ and depth $d$.
       - Transform $P$ to world space using $E_i^{-1}$.
       - Find voxel $V_k$ containing $P$.
       - Compute signed distance $s$ from $P$ to the voxel center.
       - If $|s| \le t$:
         - Clamp $s' = \text{clamp}(s, -t, t)$.
         - Update voxel:
           $$ w_{new} = w_{old} + 1 $$
           $$ tsdf_{new} = \frac{tsdf_{old} \cdot w_{old} + s' \cdot 1}{w_{new}} $$
         - Update color if enabled [S1, unofficial].

7. **Mesh Extraction**:
   - Call $M = V.\text{extract\_triangle\_mesh()}$.
   - Compute vertex normals [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import open3d as o3d

def integrate_tsdf(cameras, color_images, depth_images, voxel_size=0.02, sdf_trunc=0.04, max_depth=5.0, downsample_scale=1):
    # 1. Initialize Volume
    volume = o3d.pipelines.integration.ScalableTSDFVolume(
        voxel_length=voxel_size,
        sdf_trunc=sdf_trunc,
        color_type=o3d.pipelines.integration.TSDFVolumeColorType.RGB8
    )

    for i, cam in enumerate(cameras):
        # 2. Preprocess
        if downsample_scale > 1:
            # Downsample color and depth images
            color_img = downsample_image(color_images[i], downsample_scale)
            depth_img = downsample_image(depth_images[i], downsample_scale)
        else:
            color_img = color_images[i]
            depth_img = depth_images[i]

        if depth_img is None:
            continue

        # 3. Build Intrinsics
        if hasattr(cam, 'fx'):
            fx, fy, cx, cy = cam.fx, cam.fy, cam.cx, cam.cy
        elif hasattr(cam, 'fov_x'):
            w, h = color_img.shape[1], color_img.shape[0]
            fx = w / (2.0 * np.tan(cam.fov_x / 2.0))
            fy = h / (2.0 * np.tan(cam.fov_y / 2.0))
            cx, cy = w / 2.0, h / 2.0
        else:
            w, h = color_img.shape[1], color_img.shape[0]
            fx = fy = 0.5 * (w + h)
            cx, cy = w / 2.0, h / 2.0

        intrinsic = o3d.camera.PinholeCameraIntrinsic(
            width=w, height=h, fx=fx, fy=fy, cx=cx, cy=cy
        )

        # 4. Build Extrinsic
        if hasattr(cam, 'extrinsics'):
            extrinsic = cam.extrinsics
        elif hasattr(cam, 'w2c'):
            extrinsic = cam.w2c
        elif hasattr(cam, 'c2w'):
            extrinsic = np.linalg.inv(cam.c2w)
        else:
            # Construct from R and T
            extrinsic = np.eye(4)
            extrinsic[:3, :3] = cam.R
            extrinsic[:3, 3] = cam.T

        # 5. Create RGBD Image
        # Convert color to uint8 if needed
        if color_img.dtype != np.uint8:
            color_img = (color_img * 255).astype(np.uint8)
        
        # Convert depth to float32, handle invalids
        depth_float = depth_img.astype(np.float32)
        depth_float[depth_float > max_depth] = 0.0
        depth_float[depth_float <= 0] = 0.0 # Clamp invalids

        rgbd = o3d.geometry.RGBDImage.create_from_color_and_depth(
            o3d.geometry.Image(color_img),
            o3d.geometry.Image(depth_float),
            depth_scale=1.0,
            depth_trunc=max_depth,
            convert_rgb_to_intensity=False
        )

        # 6. Integrate
        volume.integrate(rgbd, intrinsic, extrinsic)

    # 7. Extract Mesh
    mesh = volume.extract_triangle_mesh()
    mesh.compute_vertex_normals()
    return mesh

def downsample_image(img, scale):
    # Simple nearest-neighbor or average downsampling
    h, w = img.shape[:2]
    new_h, new_w = h // scale, w // scale
    if img.ndim == 3:
        return img[::scale, ::scale, :]
    else:
        return img[::scale, ::scale]
```

**Edge Cases:**
- **Invalid Depth**: Values $> max\_depth$ or 0 are ignored. [S1, unofficial]
- **Precision**: Float32 depth is standard. [S1, unofficial]
- **Boundaries**: Voxels outside the integration bounds are untouched. [S1, unofficial]

## Operating characteristics
- **Time Complexity**: $O(N \cdot H \cdot W)$, where $N$ is the number of cameras, $H \times W$ is image resolution. Integration is linear in the number of pixels.
- **Memory Complexity**: $O(V)$, where $V$ is the number of active voxels. Sparse storage helps.
- **Performance**: Fast on modern GPUs. Downsample scale reduces memory and time significantly. [S1, unofficial]
- **Strengths**: Robust to noise, handles multi-view fusion naturally, produces watertight meshes.
- **Limitations**: Memory intensive for large volumes, truncation parameter affects surface detail.

## Background
Originated from the Open3D library's integration pipeline. The `ScalableTSDFVolume` class provides a memory-efficient way to integrate RGB-D data. It is widely used in 3D reconstruction pipelines, especially in datasets like Gaustudio. [S1, unofficial]

## Sources
[S1] GAP-LAB-CUHK-SZ/gaustudio, https://github.com/GAP-LAB-CUHK-SZ/gaustudio, fetched 2026-10-26 (unofficial)
