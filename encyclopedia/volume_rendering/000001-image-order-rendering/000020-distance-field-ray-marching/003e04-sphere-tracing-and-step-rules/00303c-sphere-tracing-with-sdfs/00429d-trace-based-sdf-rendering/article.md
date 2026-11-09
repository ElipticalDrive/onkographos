# Trace-based SDF Rendering

Also called: voxel SDF tracing

## Overview
Trace-based SDF Rendering is a technique that traces rays through a sparse voxel model using Signed Distance Fields (SDF) to determine hit depth and voxel ID, then retrieves color from the hit voxel's spherical harmonic (SH) coefficients [S1, unofficial]. It differs from standard volume rendering by relying on the geometric precision of the SDF for ray marching rather than volumetric integration, and from pure mesh rendering by storing appearance data within the voxel structure rather than on triangle faces [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: A sparse voxel model containing geometry (SDF) and appearance (SH coefficients), a camera view (origin, direction, intrinsics), and configuration parameters (`near_plane`, `vox_geo_mode`, `density_mode`) [S1, unofficial].
*   **Outputs**: A rendered color tensor, hit depth tensor, and hit voxel ID tensor [S1, unofficial].

**Data Structures**
*   **SparseVoxelModel**: Stores voxel geometry and SH coefficients.
    *   `sh_degree`: Degree of spherical harmonics (e.g., 0 for constant color) [S1, unofficial].
    *   `sh0`: Tensor of SH coefficients for the zeroth degree (base color), indexed by voxel ID [S1, unofficial].
    *   `vox_key`: Keys mapping voxel IDs to their spatial location or data [S1, unofficial].
    *   `grid_pts_xyz`: Coordinates of grid points used for TSDF fusion [S1, unofficial].
*   **Hit Data**:
    *   `hit_depth`: Float tensor storing the distance along the ray to the surface [S1, unofficial].
    *   `hit_vox_id`: Integer tensor storing the ID of the voxel hit by the ray. Value `-1` indicates no hit (background) [S1, unofficial].

**Algorithm Steps**
1.  **Ray Tracing**: For each view, invoke the voxel model's rendering function with the SDF trace mode [S1, unofficial].
    *   The system traces rays through the sparse voxel structure.
    *   It uses the SDF to determine the intersection depth (`hit_depth`) and identifies the specific voxel containing the hit point (`hit_vox_id`) [S1, unofficial].
2.  **Color Retrieval**: Retrieve the base color from the zeroth-degree SH coefficients (`sh0`) indexed by `hit_vox_id` [S1, unofficial].
    *   The color tensor is extracted and the last axis (RGB) is moved to the front to match image layout [S1, unofficial].
3.  **Masking**: Apply a boolean mask to the rendering. Pixels where `hit_vox_id` is `-1` are masked out (set to background or ignored) [S1, unofficial].
4.  **Evaluation (Optional)**: Compute Mean Squared Error (MSE) and Peak Signal-to-Noise Ratio (PSNR) against ground truth images for quality assessment [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_trace_sdf(voxel_model, view, near_plane=0.02):
    """
    Renders a view using trace-based SDF method.
    
    Args:
        voxel_model: SparseVoxelModel instance with sh0 coefficients.
        view: Camera view object containing origin and direction.
        near_plane: Minimum ray distance for visibility checks.
        
    Returns:
        rendering: Color tensor of shape (H, W, 3).
    """
    # Step 1: Trace rays through the sparse voxel SDF
    # This returns hit_depth and hit_vox_id tensors
    hit_depth, hit_vox_id = voxel_model.render_trace_sdf(view)
    
    # Step 2: Retrieve base color from SH0 coefficients
    # Index sh0 by hit_vox_id and move the color channel axis to the front
    # Assuming sh0 shape is (N_voxels, 3) or (N_voxels, 1, 3) for SH0
    color_tensor = voxel_model.sh0[hit_vox_id]
    rendering = color_tensor.moveaxis(-1, 0)
    
    # Step 3: Mask out background/non-hit pixels
    # hit_vox_id == -1 indicates no intersection
    mask = (hit_vox_id != -1)
    rendering = rendering * mask
    
    return rendering

def evaluate_quality(rendering, ground_truth):
    """
    Computes MSE and PSNR between rendered and ground truth images.
    """
    # Clip to valid range [0, 1]
    render_clipped = rendering.clip(0, 1)
    gt_clipped = ground_truth.clip(0, 1)
    
    # Compute MSE
    mse = ((render_clipped - gt_clipped) ** 2).mean()
    
    # Compute PSNR
    psnr = -10 * torch.log10(mse)
    
    return mse, psnr
```

**Parameters**
*   `near_plane`: Default value `0.02`. Used in visibility checks (`mark_max_samp_rate`, `mark_near`) to prevent near-plane artifacts [S1, unofficial].
*   `vox_geo_mode`: Configuration parameter for voxel geometry mode. Potentially overwritable via command line argument `--overwrite_vox_geo_mode` [S1, unofficial].
*   `density_mode`: Configuration parameter for density mode, printed during rendering setup [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Background Handling**: Hit voxels with ID `-1` represent background or non-intersected rays. These must be explicitly masked out by multiplying the rendering tensor by `(hit_vox_id != -1)` to avoid invalid color data [S1, unofficial].
*   **Precision**: The method relies on `float32` precision for CUDA operations [S1, unofficial].
*   **TSDF Fusion**: The underlying geometry is often derived from TSDF fusion, which requires careful handling of bandwidth and pruning thresholds to ensure valid SDF values [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Depends on the number of voxels and the efficiency of the sparse voxel traversal. The SDF step size allows for larger steps than uniform grid ray marching, potentially reducing the number of evaluations per ray [S1, unofficial].
*   **Memory Complexity**: Proportional to the number of active voxels in the sparse model. Each voxel stores SH coefficients (size depends on `sh_degree`) and geometric data [S1, unofficial].
*   **Strengths**: High-quality geometry reconstruction from multi-view data via TSDF fusion; efficient rendering using sparse structures; support for high-frequency appearance via SH coefficients [S1, unofficial].
*   **Limitations**: Requires pre-processing (TSDF fusion, voxelization); may struggle with highly transparent or thin structures if not properly captured in the SDF; memory usage can grow with scene complexity [S1, unofficial].
*   **When to Use**: Suitable for novel view synthesis from multi-view images where high-fidelity geometry and appearance are required, and the scene can be represented as a sparse voxel grid [S1, unofficial].
*   **When Not to Use**: Not suitable for real-time applications with extremely limited memory or when dynamic geometry changes require frequent re-voxelization [S1, unofficial].

## Background
Trace-based SDF Rendering is implemented in the `svraster-win10` repository by leonhardrobin [S1, unofficial]. It builds upon the principles of sphere tracing with SDFs and integrates them with sparse voxel models and spherical harmonics for appearance representation [S1, unofficial]. The method is part of a pipeline that includes TSDF fusion for geometry reconstruction and adaptive voxel subdivision [S1, unofficial].

## Sources
[S1] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
