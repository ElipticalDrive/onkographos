# TSDF Volume Integration with Uncertainty Weighting

Also called: NeRF-SLAM Fusion, Sigma-Fusion

## Overview
This method fuses depth maps into a Truncated Signed Distance Function (TSDF) volume by weighting each pixel's contribution based on its depth uncertainty (inverse standard deviation), rather than using uniform weights [S1, unofficial]. It solves the problem of noisy depth measurements degrading reconstruction quality in SLAM pipelines by down-weighting unreliable measurements [S1, unofficial]. It differs from standard TSDF fusion (e.g., [TSDF Reconstruction from Depth Maps](9982-tsdf-reconstruction-from-depth-maps.md)) by incorporating per-pixel confidence metrics derived from the depth estimator's covariance, allowing for robust integration in the presence of sensor noise or estimation errors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A SLAM packet containing camera poses (extrinsics), depth maps (stored as inverse depth), depth covariance (uncertainty), and RGB images.
- **Output:** A TSDF volume (stored in a `VoxelBlockGrid` or legacy `ScalableTSDFVolume`) containing fused geometry and color.

**Data Structures:**
The primary data structure is the Open3D `VoxelBlockGrid` (new) or `ScalableTSDFVolume` (legacy) [S1, unofficial]. The `VoxelBlockGrid` stores attributes per voxel block:
- `tsdf`: float32, 1 channel.
- `weight`: float32, 1 channel.
- `color`: float32, 3 channels (RGB) [S1, unofficial].

**Parameters:**
- `voxel_size`: Physical size of a voxel in meters. Typical value: $6.0 / 512$ meters (for a $3m \times 3m \times 3m$ room with $512^3$ grid) [S1, unofficial].
- `sdf_trunc`: Truncation distance for the TSDF in meters. Typical value: $0.10$ [S1, unofficial].
- `max_depth`: Maximum depth in meters for integration. Depths beyond this are ignored. Typical value: $6.0$ [S1, unofficial].
- `max_weight`: Maximum weight value allowed in the volume (units: $1/m^2$). Typical value: $20.0$ [S1, unofficial].
- `block_resolution`: Resolution of each voxel block in the `VoxelBlockGrid`. Typical value: $16$ [S1, unofficial].
- `block_count`: Maximum number of voxel blocks allowed in the grid. Typical value: $5000$ [S1, unofficial].
- `depth_scale`: Scale factor applied to depth values. Typical value: $1.0$ [S1, unofficial].
- `min_depth`: Minimum valid depth in meters. Typical value: $0.01$ [S1, unofficial].
- `dsf`: Down-scaling factor for camera intrinsics. Hardcoded value: $8.0$ [S1, unofficial].
- `max_depth_sigma_thresh`: Threshold for depth sigma (uncertainty). Depths with sigma higher than this are considered invalid/out of reconstruction. [S1, unofficial].
- `min_weight_for_render`: Minimum weight required for a pixel to be included in volume rendering. Typical value: $0.01$ [S1, unofficial].

**Steps:**
1. **Initialize Volume:** Create the TSDF volume using Open3D's `ScalableTSDFVolume` (legacy) or `VoxelBlockGrid` (new) [S1, unofficial].
2. **Receive Packet:** Get a SLAM packet containing camera poses, depth maps, depth covariance, and RGB images [S1, unofficial].
3. **Determine Fusion Mode:** If the mode is 'sigma', use uncertainty-based masking; otherwise, use uniform masking where all depth weights are set to 1 [S1, unofficial].
4. **Convert Depth:** Convert depth values from inverse depth (`idepth`) to linear depth by taking the reciprocal:
   $$ \text{depths} = \text{packet}['\text{cam0\_idepths\_up}'].^{\text{-1}} $$
   [S1, unofficial]
5. **Compute Weights:** Calculate integration weights from depth covariance ($\sigma^2$). The weight is the inverse square root of the covariance:
   $$ w = \frac{1}{\sqrt{\text{covariance}}} $$
   Specifically:
   $$ \text{weights} = \text{packet}['\text{cam0\_depths\_cov\_up}'].^{\text{-0.5}} $$
   Higher standard deviation results in lower weight [S1, unofficial].
6. **Masking:** Compute depth masks based on uncertainty (if sigma mode) or uniform criteria to identify valid pixels. Pixels with sigma > `max_depth_sigma_thresh` are masked out [S1, unofficial].
7. **Convert to Open3D:** Convert PyTorch tensors to Open3D tensors using DLpack to avoid memory copies:
   ```python
   o3d_depth = o3d.t.geometry.Image(o3d.core.Tensor.from_dlpack(torch.utils.dlpack.to_dlpack(depth)))
   ```
   [S1, unofficial]
8. **Block Activation:** For each camera frame, compute the unique voxel block coordinates intersected by the camera frustum using the depth image, intrinsics, extrinsics, depth scale, and `max_depth` [S1, unofficial].
9. **Activate Blocks:** Activate the identified blocks in the volume's hash map [S1, unofficial].
10. **Find Indices:** Find the indices and masks for the activated blocks to prepare for integration [S1, unofficial].
11. **Synchronize:** Synchronize the CUDA stream to ensure completion of volume operations [S1, unofficial].
12. **Integrate:**
    - If using the legacy volume (`use_old_volume` is True), create an Open3D RGBD image from color and depth, then integrate it into the volume using the intrinsic and extrinsic matrices [S1, unofficial].
    - If using the new volume (`use_old_volume` is False), call `custom_volume_integrate`, which handles block activation and index finding, implying further integration steps are handled by the Open3D backend [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integrate_frame(packet, volume, use_sigma=True, use_old_volume=False):
    # 1. Convert inverse depth to linear depth
    # packet['cam0_idepths_up'] is a PyTorch tensor
    idepth = packet['cam0_idepths_up']
    depths = idepth.pow(-1)
    
    # 2. Compute weights based on covariance
    # packet['cam0_depths_cov_up'] is a PyTorch tensor
    cov = packet['cam0_depths_cov_up']
    if use_sigma:
        # Inverse square root of covariance
        weights = cov.pow(-0.5)
    else:
        weights = torch.ones_like(depths)
        
    # 3. Masking based on uncertainty threshold
    # max_depth_sigma_thresh is a parameter
    sigma = cov.sqrt()
    valid_mask = sigma < max_depth_sigma_thresh
    
    # Also mask by min/max depth
    depth_valid = (depths > min_depth) & (depths < max_depth)
    valid_mask = valid_mask & depth_valid
    
    # 4. Convert to Open3D format
    # Using DLpack for zero-copy transfer
    o3d_depth = o3d.t.geometry.Image(
        o3d.core.Tensor.from_dlpack(torch.utils.dlpack.to_dlpack(depths))
    )
    o3d_color = o3d.t.geometry.Image(
        o3d.core.Tensor.from_dlpack(torch.utils.dlpack.to_dlpack(packet['cam0_rgb_up']))
    )
    o3d_weight = o3d.t.geometry.Image(
        o3d.core.Tensor.from_dlpack(torch.utils.dlpack.to_dlpack(weights))
    )
    
    # 5. Compute intersected blocks
    try:
        block_coords = compute_unique_block_coordinates(
            o3d_depth, o3d_color, o3d_weight,
            intrinsics, extrinsics, 
            voxel_size, sdf_trunc, max_depth, dsf
        )
    except Exception as e:
        # Exception if parameters are invalid (e.g., max_depth issues)
        print(f"Integration failed: {e}")
        return False
        
    # 6. Activate blocks
    volume.activate_blocks(block_coords)
    
    # 7. Find indices and masks
    indices, masks = volume.find_block_indices_and_masks(block_coords)
    
    # 8. Synchronize CUDA
    torch.cuda.synchronize()
    
    # 9. Integrate
    if use_old_volume:
        # Legacy path: Create RGBD image and integrate
        rgbd = o3d.t.geometry.RGBDImage(o3d_color, o3d_depth)
        volume.integrate(rgbd, intrinsics, extrinsics)
    else:
        # New path: Custom integration
        custom_volume_integrate(volume, o3d_depth, o3d_color, o3d_weight, 
                                intrinsics, extrinsics, indices, masks)
                                
    return True
```

## Subtopics

### [Adaptive-resolution Octree TSDF Integration](17869-adaptive-resolution-octree-tsdf-integration.md)
A volumetric SLAM pipeline that integrates and renders depth images at an adaptive level of detail to address aliasing, reconstruction quality, and efficiency simultaneously. It differs from fixed-resolution grids by dynamically selecting the integration scale based on the effective sensor resolution relative to the distance from the scene, and from other octree methods by using a specific mipmapped block structure that balances multi-scale representation with spatial locality. It builds on [TSDF Volume Integration with Uncertainty Weighting](15425-tsdf-volume-integration-with-uncertainty-weighting.md) by replacing uniform or uncertainty-weighted fusion with adaptive-resolution fusion.

## Operating characteristics
- **Time Complexity:** Depends on the number of active voxel blocks and the resolution of the depth map. The block activation step is $O(N)$ where $N$ is the number of pixels, but only active blocks are processed.
- **Memory Complexity:** Proportional to the number of active voxel blocks. The `VoxelBlockGrid` limits this with `block_count` (e.g., 5000 blocks) [S1, unofficial].
- **Measured Performance:** The pipeline is GPU memory intensive, particularly due to frame-to-frame correlation volumes and volumetric rendering [S1, unofficial].
- **Strengths:** Robust to noisy depth measurements by down-weighting high-uncertainty pixels [S1, unofficial]. Efficient block-based activation avoids processing empty space.
- **Limitations:** High GPU memory usage. Requires careful tuning of `max_depth_sigma_thresh` and `max_weight` [S1, unofficial].
- **When to use:** When depth estimates have associated uncertainty metrics (e.g., from a neural depth estimator) and noise is a significant factor.
- **When not to use:** When computational resources are severely limited or when depth data is uniformly reliable (uniform weighting may suffice).

## Background
Originated in the NeRF-SLAM project by Rosinol, Leonard, and Carlone (2022) [S1, unofficial]. The method was introduced to enable real-time dense monocular SLAM with Neural Radiance Fields, specifically addressing the need for robust volumetric integration in the presence of uncertain depth predictions from neural networks [S1, unofficial]. It is implemented in the NeRF-SLAM repository [S1, unofficial].

## Sources
[S1] ToniRV/NeRF-SLAM, https://github.com/ToniRV/NeRF-SLAM, fetched 2026-10-26 (unofficial)
