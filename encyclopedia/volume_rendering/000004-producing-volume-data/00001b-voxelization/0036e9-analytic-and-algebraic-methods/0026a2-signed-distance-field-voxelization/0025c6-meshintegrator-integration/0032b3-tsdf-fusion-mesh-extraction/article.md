# TSDF Fusion Mesh Extraction

Also called: Bounded TSDF Mesh Extraction, ScalableTSDFVolume Mesh Extraction, TSDF Volume Integration

## Overview
This method extracts a triangle mesh from a Truncated Signed Distance Function (TSDF) volume that has been integrated with color information from 2D Gaussian Splatting depth and color maps. It solves the problem of converting volumetric depth representations into explicit surface geometry. It differs from unbounded extraction by operating within a fixed depth range and voxel resolution, making it suitable for bounded scenes where memory usage is constrained by the `depth_trunc` parameter.

## How it works
**Inputs:**
- A list of viewpoint cameras (intrinsic and extrinsic matrices).
- RGB and depth maps for each viewpoint.
- Optional background masks (if available).
- Parameters: `voxel_size`, `sdf_trunc`, `depth_trunc`.

**Outputs:**
- A triangle mesh object (e.g., Open3D `TriangleMesh`).

**Data Structures:**
- `ScalableTSDFVolume`: An Open3D data structure storing the TSDF values and weights. It is initialized with a specific voxel length and truncation distance.
- `RGBDImage`: An Open3D structure combining an RGB image (uint8) and a depth image (float).

**Steps:**

1.  **Initialize Volume:** Create a `ScalableTSDFVolume` instance.
    - Set `voxel_length` to `voxel_size`.
    - Set `sdf_trunc` to `sdf_trunc`.
    - Set `color_type` to `RGB8` to store color information.

2.  **Process Viewpoints:** Iterate through each camera in the `viewpoint_stack`.
    - **Masking:** If `mask_background` is True and a ground truth alpha mask exists, set depth values to 0 where the alpha value is less than 0.5. This removes background noise.
    - **Create RGBDImage:**
        - Clip RGB values to the range $[0, 1]$.
        - Multiply by 255 and cast to `uint8`.
        - Scale depth by 1.0 (assuming depth is already in meters or consistent units).
        - Create the `RGBDImage` using the RGB and processed depth maps, passing `depth_trunc` to the creator to handle out-of-range depths.
    - **Integrate:** Call the integration method on the `ScalableTSDFVolume` with the `RGBDImage`, camera intrinsic matrix, and camera extrinsic matrix. This updates the TSDF and weight fields for voxels within the truncation band.

3.  **Extract Mesh:** Call the mesh extraction method on the integrated `ScalableTSDFVolume`. This typically uses a marching cubes algorithm to find the zero-level set of the TSDF.

4.  **Return:** Return the resulting triangle mesh.

**Parameters:**
- `voxel_size`: Resolution of the TSDF volume. Default: 0.004.
- `sdf_trunc`: Truncation distance for the SDF. Default: 0.02.
- `depth_trunc`: Maximum depth range for integration. Default: 3.0.
- `mask_background`: Whether to mask background using alpha masks. Default: True.

**Edge Cases and Pitfalls:**
- **Depth Clipping:** Depths outside `depth_trunc` are ignored or handled by the RGBDImage creator. Ensure depth maps are scaled correctly.
- **Voxel Resolution:** If `voxel_size` is too large, fine details are lost. If too small, memory usage increases significantly.
- **Truncation:** `sdf_trunc` determines the width of the truncation band. Too small leads to noisy surfaces; too large leads to blurry surfaces.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import open3d as o3d
import numpy as np

def extract_mesh_bounded(cameras, rgb_maps, depth_maps, masks=None, 
                         voxel_size=0.004, sdf_trunc=0.02, depth_trunc=3.0, 
                         mask_background=True):
    """
    Extract mesh from TSDF volume integrated from 2D Gaussian Splatting outputs.
    """
    # Initialize ScalableTSDFVolume
    # voxel_length: size of one voxel in meters
    # sdf_trunc: truncation distance
    # color_type: RGB8 for 8-bit color
    tsdf_vol = o3d.pipelines.integration.ScalableTSDFVolume(
        voxel_length=voxel_size,
        sdf_trunc=sdf_trunc,
        color_type=o3d.pipelines.integration.TSDFVolumeColorType.RGB8
    )
    
    for i, camera in enumerate(cameras):
        rgb = rgb_maps[i]
        depth = depth_maps[i]
        
        # Handle background masking
        if mask_background and masks is not None and masks[i] is not None:
            alpha_mask = masks[i]
            # Set depth to 0 where alpha < 0.5
            depth[alpha_mask < 0.5] = 0.0
            
        # Prepare RGB image: clip to [0,1], multiply by 255, cast to uint8
        rgb_uint8 = np.clip(rgb, 0.0, 1.0) * 255.0
        rgb_uint8 = rgb_uint8.astype(np.uint8)
        
        # Create RGBDImage
        # depth_scale is 1.0 as per notes
        rgbd_image = o3d.geometry.RGBDImage.create_from_color_and_depth(
            o3d.geometry.Image(rgb_uint8),
            o3d.geometry.Image(depth.astype(np.float32)),
            depth_trunc=depth_trunc,
            depth_scale=1.0,
            convert_rgb_to_intensity=False
        )
        
        # Integrate into volume
        # camera.intrinsic and camera.extrinsic are assumed to be available
        tsdf_vol.integrate(
            rgbd_image,
            intrinsic=camera.intrinsic,
            extrinsic=camera.extrinsic
        )
        
    # Extract mesh
    mesh = tsdf_vol.extract_triangle_mesh()
    
    return mesh
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and the number of viewpoints. Integration is the dominant cost.
- **Memory Complexity:** Proportional to the volume of the bounding box defined by `depth_trunc` and the voxel resolution. `ScalableTSDFVolume` manages memory efficiently by only allocating blocks that are integrated.
- **Strengths:** Robust to noise due to truncation; integrates color information; scalable memory usage.
- **Limitations:** Requires a bounded scene or a defined `depth_trunc`; resolution is limited by voxel size; may produce noisy meshes if integration is sparse.
- **When to use:** When extracting meshes from depth maps with known camera poses, especially in 3D reconstruction pipelines like 2D Gaussian Splatting.
- **When not to use:** For unbounded scenes or when high-frequency details are required beyond the voxel resolution.

## Background
This method is commonly used in 3D reconstruction pipelines, particularly those involving 2D Gaussian Splatting, to convert the volumetric representation into a triangle mesh. It leverages Open3D's `ScalableTSDFVolume` for efficient integration and extraction.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
