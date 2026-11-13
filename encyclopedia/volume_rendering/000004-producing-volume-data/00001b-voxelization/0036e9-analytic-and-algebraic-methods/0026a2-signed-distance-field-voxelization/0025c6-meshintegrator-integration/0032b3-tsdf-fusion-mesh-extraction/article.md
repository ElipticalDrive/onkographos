# TSDF Fusion Mesh Extraction

Also called: Bounded TSDF Mesh Extraction, Direct extraction, Progressive extraction, ScalableTSDFVolume Mesh Extraction, Sparse Voxel Mesh Extraction, TSDF Volume Integration

## Overview
TSDF Fusion Mesh Extraction reconstructs a 3D triangle mesh by integrating multi-view depth and color observations into a Truncated Signed Distance Function (TSDF) volume and extracting the surface via marching cubes [S1, unofficial][S2, unofficial]. It converts per-surfel depth outputs into a unified 3D mesh by computing a robust depth value per pixel and fusing the resulting depth maps via TSDF [S2, unofficial]. It differs from direct neural rendering mesh extraction by relying on explicit geometric fusion rather than differentiable surface optimization [S1, unofficial]. It builds on [MeshIntegrator Integration](3630-meshintegrator-integration.md), which writes the raw distance field data, by consuming that data to produce a final watertight mesh.

## How it works
**Inputs:**
- A set of $N$ camera views, each providing an RGB image, a depth map, and optionally an alpha mask.
- Camera intrinsics (focal length, principal point) and extrinsics (rotation, translation) for each view.
- Parameters: `voxel_size`, `sdf_trunc`, `depth_trunc`, `mask_background`, `progressive_mode`, `init_lv`, `final_lv`, `bandwidth_vox`, `alpha_thres`.

**Outputs:**
- A triangle mesh (vertices and faces) with optional vertex colors.

**Data Structures:**
- **ScalableTSDFVolume**: A sparse octree-based volume structure.
  - **Voxel**: Stores a truncated signed distance value (float) and an accumulated weight (float).
  - **Octree Level**: Controls resolution. Level $L$ corresponds to a voxel size of $2^{-L} \times \text{base\_voxel\_size}$ (conceptually; implementation uses `voxel_length` as the base unit).
  - **Grid Points**: The spatial coordinates of voxel centers, accessed via `voxel_model.grid_pts_xyz` in sparse variants.

**Parameters and Typical Values:**
- `voxel_size`: Base voxel length in world units. Default: 0.004 [S1, unofficial][S3, unofficial].
- `sdf_trunc`: Truncation distance in world units. Voxels with distance $> sdf\_trunc$ are clamped. Default: 0.02 [S1, unofficial][S3, unofficial]. Often set to $5.0 \times \text{voxel\_size}$ [S1, unofficial].
- `depth_trunc`: Maximum depth range for integration in world units. Depth values $> depth\_trunc$ are ignored. Default: 3.0 [S1, unofficial][S3, unofficial].
- `alpha_thres`: Alpha threshold for masking. Pixels with alpha $< \alpha\_thres$ are treated as background. Default: 0.5 [S2, unofficial].
- `init_lv`: Starting octree level for progressive extraction. Default: 8 [S2, unofficial].
- `final_lv`: Target octree level for extraction. Levels above this are clamped in non-progressive mode. Default: 10 [S2, unofficial].
- `bandwidth_vox`: Truncation bandwidth in voxel size units. Default: 5.0 [S2, unofficial].

**Steps:**

1. **Initialize Volume:**
   Create a `ScalableTSDFVolume` with the specified `voxel_length`, `sdf_trunc`, and `color_type` (typically `RGB8` for vertex colors) [S3, unofficial].

2. **Prepare Input Data:**
   For each camera view $i$:
   a. **Masking:** If `mask_background` is enabled and a ground truth alpha mask exists, set depth values to 0 where the alpha mask $< \alpha\_thres$ [S1, unofficial][S3, unofficial].
   b. **RGBD Conversion:** Create an Open3D `RGBDImage` from the RGB and depth maps.
      - Clip RGB values to $[0, 1]$.
      - Multiply by 255 and cast to `uint8`.
      - Set `depth_scale` to 1.0.
      - Set `depth_trunc` to ignore values outside the valid range [S3, unofficial].

3. **Integration:**
   For each prepared `RGBDImage` and corresponding camera intrinsic/extrinsic matrices:
   - Call `volume.integrate(rgbd, intrinsic, extrinsic)` [S3, unofficial].
   - This step overwrites voxel values within the truncation band. Voxels within `sdf_trunc` get distance clamped to $\pm sdf\_trunc$ and weight updated. Voxels outside the band in those blocks get `tsdf = 0, weight = 0` [Parent].

4. **Mesh Extraction:**
   Depending on the mode:
   - **Non-Progressive (Direct):**
     - Call `extract_mesh(args, data_pack, voxel_model, final_lv, crop_bbox)` [S2, unofficial].
     - Octree levels strictly greater than `outside_level + final_lv` are clamped [S2, unofficial].
   - **Progressive:**
     - Call `extract_mesh_progressive(args, data_pack, voxel_model, init_lv, final_lv, crop_bbox)` [S2, unofficial].
     - The initial extraction level is marked with `>>> Init level <<<` when `lv == outside_level + init_lv` [S2, unofficial].

5. **Post-Processing (Optional):**
   - Extract connected clusters.
   - Keep the largest $N$ clusters (default $N=50$) [S1, unofficial].
   - Save as PLY or OBJ.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import open3d as o3d
import numpy as np

def extract_mesh_from_rgbd(
    cameras,
    voxel_size=0.004,
    sdf_trunc=0.02,
    depth_trunc=3.0,
    alpha_thres=0.5,
    mask_background=True,
    progressive=False,
    init_lv=8,
    final_lv=10
):
    """
    Extract a mesh from a set of RGBD images using TSDF fusion.
    
    Args:
        cameras: List of dicts with keys 'rgb', 'depth', 'intrinsic', 'extrinsic', 'gt_alpha_mask' (optional).
        voxel_size: Base voxel size.
        sdf_trunc: Truncation distance.
        depth_trunc: Max depth range.
        alpha_thres: Alpha threshold for background masking.
        mask_background: Whether to use alpha mask.
        progressive: Whether to use progressive extraction.
        init_lv: Initial octree level for progressive mode.
        final_lv: Final octree level.
        
    Returns:
        o3d.geometry.TriangleMesh
    """
    # 1. Initialize Volume
    # color_type=RGB8 stores vertex colors
    volume = o3d.pipelines.integration.ScalableTSDFVolume(
        voxel_length=voxel_size,
        sdf_trunc=sdf_trunc,
        color_type=o3d.pipelines.integration.TSDFVolumeColorType.RGB8
    )
    
    # 2. Integrate each view
    for cam in cameras:
        rgb = cam['rgb']
        depth = cam['depth']
        intrinsic = cam['intrinsic']
        extrinsic = cam['extrinsic']
        alpha_mask = cam.get('gt_alpha_mask', None)
        
        # 2a. Masking
        if mask_background and alpha_mask is not None:
            # Set depth to 0 where alpha < threshold
            # Note: Open3D integration ignores depth=0 if depth_trunc > 0, 
            # but explicit masking ensures clean background.
            depth[alpha_mask < alpha_thres] = 0.0
            
        # 2b. Create RGBD Image
        # Clip RGB to [0,1], scale to [0,255], cast to uint8
        rgb_uint8 = np.clip(rgb, 0.0, 1.0) * 255.0
        rgb_uint8 = rgb_uint8.astype(np.uint8)
        
        rgbd = o3d.geometry.RGBDImage.create_from_color_and_depth(
            o3d.geometry.Image(rgb_uint8),
            o3d.geometry.Image(depth.astype(np.float32)),
            depth_trunc=depth_trunc,
            depth_scale=1.0,
            convert_rgb_to_intensity=False
        )
        
        # 2c. Integrate
        volume.integrate(
            rgbd,
            intrinsic=intrinsic,
            extrinsic=extrinsic
        )
        
    # 3. Extract Mesh
    if progressive:
        # Progressive extraction requires specific args structure or internal state
        # Assuming a helper function exists that handles the octree traversal
        mesh = extract_mesh_progressive(
            init_lv=init_lv,
            final_lv=final_lv,
            voxel_model=volume # Simplified reference
        )
    else:
        # Direct extraction
        mesh = volume.extract_triangle_mesh()
        
    return mesh

def extract_mesh_progressive(init_lv, final_lv, voxel_model):
    """
    Placeholder for progressive extraction logic.
    In practice, this traverses the octree from init_lv to final_lv.
    """
    # Implementation detail: Iterate octree levels from init_lv to final_lv.
    # For levels > final_lv, clamp values.
    # Return merged mesh.
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating point precision in TSDF accumulation can lead to artifacts if `voxel_size` is too small relative to scene scale.
- **Boundaries:** Voxels outside the `depth_trunc` range are ignored. Ensure `depth_trunc` covers the entire object of interest.
- **Degenerate Input:** If `alpha_mask` is missing but `mask_background` is True, the method may fail or produce noisy results. [G] Check for mask existence before masking.
- **Clamping:** In non-progressive mode, octree levels strictly greater than `outside_level + final_lv` are clamped, indicated by the suffix `>>> Level clamped <<<` in logs [S2, unofficial].

## Subtopics

### [TSDF Reconstruction from Depth Maps](9982-tsdf-reconstruction-from-depth-maps.md)
TSDF Reconstruction from Depth Maps is a method to reconstruct a 3D mesh from multi-view depth images by integrating them into a Truncated Signed Distance Function (TSDF) volume and extracting the surface via marching cubes. It converts per-surfel depth outputs into a unified 3D mesh by computing a robust depth value per pixel and fusing the resulting depth maps via TSDF. It differs from direct neural rendering mesh extraction by relying on explicit geometric fusion rather than differentiable surface optimization.

### SDF and RGB Fusion (Fuser)
A variant that fuses SDF and RGB data using specific bandwidth parameters.
- **Parameters:**
  - `xyz`: Input coordinates, set to `voxel_model.grid_pts_xyz` [S2, unofficial].
  - `bandwidth`: Set to 20 times the minimum voxel size: `voxel_model.vox_size.min().item() * 20` [S2, unofficial].
  - `use_trunc`: Boolean flag, set to `True` for truncation [S2, unofficial].
  - `fuse_tsdf`: Boolean flag, set to `True` to fuse Truncated SDF [S2, unofficial].
- **Pitfalls:**
  - Alternative bandwidth calculation is commented out, which would use the maximum voxel size repeated 8 times per voxel key, multiplied by 3 [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Integration is $O(N \cdot V)$, where $N$ is the number of views and $V$ is the number of voxels in the truncation band. Extraction is $O(M)$, where $M$ is the number of voxels in the volume.
- **Memory Complexity:** $O(V)$, where $V$ is the total number of voxels in the sparse octree. ScalableTSDFVolume uses sparse storage, so memory usage depends on the occupied volume.
- **Measured Performance:** With `voxel_size=0.004`, `sdf_trunc=0.02`, and `depth_trunc=3.0`, integration is efficient for scenes up to several meters in size [S1, unofficial][S3, unofficial].
- **Strengths:** Robust to noise, handles multi-view consistency well, produces watertight meshes.
- **Limitations:** Requires accurate depth maps. Performance degrades with very large scenes if not using sparse structures. Progressive extraction can be slower due to octree traversal.
- **When to Use:** When high-quality, watertight meshes are needed from multi-view depth data.
- **When Not to Use:** When real-time extraction is required, or when depth data is unreliable.

## Background
Originating from the integration of multi-view stereo techniques, TSDF fusion has been widely used in 3D reconstruction. The `ScalableTSDFVolume` implementation in Open3D provides a sparse, efficient way to handle large volumes. It is commonly used in 3D Gaussian Splatting pipelines for mesh extraction [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
