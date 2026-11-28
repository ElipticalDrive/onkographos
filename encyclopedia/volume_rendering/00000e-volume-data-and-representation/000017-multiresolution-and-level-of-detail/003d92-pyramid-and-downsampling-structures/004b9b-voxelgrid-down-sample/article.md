# VoxelGrid Down Sample

Also called: Cascade Repacking, RepackAndUpscaleKernel, VoxelGrid Up Sample

## Overview
VoxelGrid Down Sample reduces the resolution of a voxel grid representation to generate lower-resolution levels of a multi-resolution hierarchy [S1, unofficial][S2, unofficial]. It addresses memory and computational cost constraints in volume rendering by creating coarser approximations of scene geometry or density [S3, unofficial]. It differs from sibling methods like [Morphological Pyramid](1641-morphological-pyramid.md) by employing hardware-accelerated compute kernels that combine repacking with upsampling logic, or by using max-pooling for feature reduction, whereas [Mipmap Generation](2602-mipmap-generation.md) typically focuses on texture filtering [S2, unofficial][S4, unofficial].

## How it works
The method manifests in three distinct implementations depending on the data type (scalar/vector volume, point cloud, or deep learning feature map) and the target hardware (CPU/GPU compute shader) [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial].

### Inputs and Outputs
*   **Input:** A voxel grid (scalar or vector), a point cloud, or a feature tensor [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial].
*   **Output:** A down-sampled voxel grid, a sparse point cloud, or a reduced feature tensor [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial].

### Data Structures
1.  **Voxel Grid (Scalar/Vector):** A 3D texture (`CVCT_tex3d`) or a flat compute buffer (`cb_gv`) [S2, unofficial]. The grid has a resolution $N \times N \times N$ [S2, unofficial].
2.  **Point Cloud:** An $N \times D$ tensor of coordinates [S3, unofficial].
3.  **Feature Tensor:** A 4D tensor (Channels, Height, Width, Depth) or 3D tensor [S4, unofficial].

### Implementation 1: GPU Compute Shader (Cascade Repacking)
This implementation, found in [S2, unofficial], repacks voxel data from a flat buffer into a 3D texture while simultaneously upsampling by a factor of 2. It is used to build cascaded levels of a Voxel Cone Tracing (VCT) system [S2, unofficial].

**Parameters:**
*   $GridResolution$: Integer resolution of the source grid [S2, unofficial].
*   $GridHalfResolution$: $GridResolution / 2$ [S2, unofficial].
*   $GridCascadeBase$: A 2D vector where $x$ is the current cascade index $\times GridResolution$ and $y$ is the previous cascade index $\times GridResolution$, used to offset the Y coordinate in the 3D texture [S2, unofficial].

**Steps:**
1.  **Thread Mapping:** For each thread with ID $id = (x, y, z)$, calculate the source coordinate in the lower-resolution grid:
    $$id1 = id \times 2 - GridHalfResolution$$ [S2, unofficial].
2.  **Boundary Check:** If any component of $id1$ is $\ge GridResolution$, the thread is out of bounds for the source data [S2, unofficial].
    *   **Fallback:** Read directly from the source flat buffer `cb_gv` using the linear index:
        $$src\_index = (id.z \times GridResolution + id.y) \times GridResolution + id.x$$
        $$f = CVCT\_gv[src\_index]$$ [S2, unofficial].
    *   Skip the upsampling logic.
3.  **Upsampling (if in bounds):**
    *   Adjust Y-coordinate for cascade offset:
        $$id1.y = id1.y + GridCascadeBase.y$$ [S2, unofficial].
    *   Sample 8 neighboring voxels from the 3D texture `CVCT_tex3d` at $id1$ and its neighbors. The neighbors are defined by offsets $(0,0,1), (1,0,0), (1,0,1), (0,1,0), (0,1,1), (1,1,0), (1,1,1)$ [S2, unofficial].
    *   Apply component-wise minimum to pairs to determine the value:
        $$ff.x = \min(CVCT\_tex3d[id1], CVCT\_tex3d[id1 + (0,0,1)])$$
        $$ff.y = \min(CVCT\_tex3d[id1 + (1,0,0)], CVCT\_tex3d[id1 + (1,0,1)])$$
        $$ff.z = \min(CVCT\_tex3d[id1 + (0,1,0)], CVCT\_tex3d[id1 + (0,1,1)])$$
        $$ff.w = \min(CVCT\_tex3d[id1 + (1,1,0)], CVCT\_tex3d[id1 + (1,1,1)])$$ [S2, unofficial].
    *   Invert the values (assuming the texture stores inverse density/opacity):
        $$ff = 1 - ff$$ [S2, unofficial].
    *   Average the four components. Note: Sources [S2, unofficial] show a discrepancy in the averaging weights. One version uses $0.25$ (commented out), and the active version uses $0.5$:
        $$f = ff.x \times 0.5 + ff.y \times 0.5 + ff.z \times 0.5 + ff.w \times 0.5$$ [S2, unofficial].
4.  **Write Output:**
    *   Adjust Y-coordinate for write offset:
        $$id.y = id.y + GridCascadeBase.x$$ [S2, unofficial].
    *   Write the inverted result back to the 3D texture:
        $$CVCT\_tex3d[id] = 1 - f$$ [S2, unofficial].

### Implementation 2: Point Cloud Voxelization
This implementation, found in [S3, unofficial], downsamples a point cloud by assigning points to voxels and returning the centers of occupied voxels. It is faster than Open3D's implementation but uses more memory by processing all points in parallel [S3, unofficial].

**Parameters:**
*   $points$: $N \times D$ tensor of point coordinates [S3, unofficial].
*   $resolution$: Float, the voxel size [S3, unofficial].
*   $range\_per\_dim$: Optional bounds array. If None, calculated as $[min(points) - 2 \times resolution, max(points) + 2 \times resolution]$ [S3, unofficial].
*   $ignore\_flat\_dim$: Boolean. If True and the Z-range is zero, treats the data as 2D [S3, unofficial].

**Steps:**
1.  **Handle Empty Input:** If $points.shape[0] == 0$, return points [S3, unofficial].
2.  **Handle Flat Dimension:** If $ignore\_flat\_dim$ is True and $range\_per\_dim[-1][0] == range\_per\_dim[-1][1]$:
    *   Set $flat\_z = True$ [S3, unofficial].
    *   Store $flat\_z\_val = range\_per\_dim[-1][0]$ [S3, unofficial].
    *   Remove the last dimension from $range\_per\_dim$ and $points$ [S3, unofficial].
3.  **Create Grid:** Create a boolean VoxelGrid with resolution $resolution$ and bounds $range\_per\_dim$ [S3, unofficial].
4.  **Mark Voxels:** Set `voxel[points] = 1` for all input points [S3, unofficial].
5.  **Retrieve Centers:** Extract the centers of all occupied voxels (`pts`) [S3, unofficial].
6.  **Restore Flat Dimension:** If $flat\_z$ was True, append a column of $flat\_z\_val$ to `pts` along the last dimension [S3, unofficial].
7.  **Return:** Return `pts` [S3, unofficial].

### Implementation 3: Max-Pooling Layer (Deep Learning)
This implementation, found in [S4, unofficial], downsamples feature maps in a neural network encoder. It applies a max filter to non-overlapping sub-regions of the input to downsample the input and reduce parameters in subsequent layers [S4, unofficial].

**Parameters:**
*   Filter Size: $2 \times 2$ [S4, unofficial].
*   Stride: 2 [S4, unofficial].
*   Applied to channels with sizes 64, 32, and 8 [S4, unofficial].

**Steps:**
1.  Divide the input volume into non-overlapping $2 \times 2$ sub-regions [S4, unofficial].
2.  For each sub-region, compute the maximum value across all channels (or per-channel if specified) [S4, unofficial].
3.  Output the maximum values as the downsampled feature map [S4, unofficial].

### Pseudocode (Unified Python-like Style)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def voxel_grid_downsample_max_pool(data: np.ndarray, factor: int) -> np.ndarray:
    """
    Downsamples a 3D voxel grid using max pooling.
    Based on [S1, unofficial].
    """
    # Validate divisibility
    if data.shape[0] % factor != 0 or data.shape[1] % factor != 0 or data.shape[2] % factor != 0:
        raise ValueError("Resolution must be divisible by factor")
    
    # Reshape and pool (equivalent to block_reduce with np.max)
    reshaped = data.reshape(
        data.shape[0] // factor, factor,
        data.shape[1] // factor, factor,
        data.shape[2] // factor, factor
    )
    new_data = reshaped.max(axis=(1, 3, 5))
    return new_data

def point_cloud_voxel_downsample(points: torch.Tensor, resolution: float, 
                                 range_per_dim: np.ndarray = None, 
                                 ignore_flat_dim: bool = False) -> torch.Tensor:
    """
    Downsamples a point cloud to voxel centers.
    Based on [S3, unofficial].
    """
    if points.shape[0] == 0:
        return points
    
    # Determine bounds
    if range_per_dim is None:
        min_pts = points.min(dim=0)[0]
        max_pts = points.max(dim=0)[0]
        range_per_dim = np.array([
            [min_pts[0] - 2 * resolution, max_pts[0] + 2 * resolution],
            [min_pts[1] - 2 * resolution, max_pts[1] + 2 * resolution],
            [min_pts[2] - 2 * resolution, max_pts[2] + 2 * resolution]
        ])
    
    flat_z = False
    flat_z_val = 0.0
    
    # Handle flat dimension
    if ignore_flat_dim and range_per_dim[-1][0] == range_per_dim[-1][1]:
        flat_z = True
        flat_z_val = range_per_dim[-1][0]
        range_per_dim = range_per_dim[:-1]
        points = points[:, :-1]
        
    # Explicit grid creation and marking to avoid undefined VoxelGrid helper
    grid_dims = np.ceil((range_per_dim[:, 1] - range_per_dim[:, 0]) / resolution).astype(int)
    grid = torch.zeros(grid_dims, dtype=torch.bool, device=points.device)
    
    # Map points to grid indices
    indices = ((points - range_per_dim[:, 0]) / resolution).long()
    indices = torch.clamp(indices, 0, grid_dims - 1)
    grid[indices[:, 0], indices[:, 1], indices[:, 2]] = True
    
    # Retrieve centers of occupied voxels
    occupied_indices = torch.nonzero(grid, as_tuple=True)
    if len(occupied_indices[0]) == 0:
        return torch.empty((0, points.shape[1]), device=points.device)
        
    pts = (torch.stack(occupied_indices, dim=1).float() + 0.5) * resolution + range_per_dim[:, 0]
    
    if flat_z:
        ones = torch.ones(pts.shape[0], 1, device=pts.device)
        flat_col = ones * flat_z_val
        pts = torch.cat([pts, flat_col], dim=-1)
        
    return pts

def compute_shader_upsample_logic(id: tuple, GridResolution: int, 
                                  GridHalfResolution: int, 
                                  GridCascadeBase: tuple,
                                  CVCT_tex3d: dict, CVCT_gv: list) -> float:
    """
    Logic for RepackAndUpscaleKernel.
    Based on [S2, unofficial].
    """
    x, y, z = id
    id1 = (x * 2 - GridHalfResolution, y * 2 - GridHalfResolution, z * 2 - GridHalfResolution)
    
    # Check bounds
    if not (id1[0] < GridResolution and id1[1] < GridResolution and id1[2] < GridResolution):
        # Fallback
        src_index = (z * GridResolution + y) * GridResolution + x
        f = CVCT_gv[src_index]
        return 1.0 - f
    
    # Upsampling
    id1_y_adj = id1[1] + GridCascadeBase[1]
    id1 = (id1[0], id1_y_adj, id1[2])
    
    # Sample 8 neighbors
    def sample(coord):
        cx, cy, cz = coord
        return CVCT_tex3d[(cx, cy, cz)]
    
    ff_x = min(sample(id1), sample((id1[0], id1[1], id1[2] + 1)))
    ff_y = min(sample((id1[0] + 1, id1[1], id1[2])), sample((id1[0] + 1, id1[1], id1[2] + 1)))
    ff_z = min(sample((id1[0], id1[1] + 1, id1[2])), sample((id1[0], id1[1] + 1, id1[2] + 1)))
    ff_w = min(sample((id1[0] + 1, id1[1] + 1, id1[2])), sample((id1[0] + 1, id1[1] + 1, id1[2] + 1)))
    
    ff = (1.0 - ff_x, 1.0 - ff_y, 1.0 - ff_z, 1.0 - ff_w)
    
    # Average with 0.5 weights
    f = ff[0] * 0.5 + ff[1] * 0.5 + ff[2] * 0.5 + ff[3] * 0.5
    
    # Write back logic simulation
    write_y = y + GridCascadeBase[0]
    # CVCT_tex3d[(x, write_y, z)] = 1.0 - f
    return 1.0 - f
```

## Subtopics

### [Volume Resampling and Overflow Guard](1772-volume-resampling-and-overflow-guard.md)
This method resamples a candidate volume grid to a target shape while enforcing a maximum voxel count to prevent downstream memory overflow. It differs from sibling methods by combining a strict safety cap with precise trilinear interpolation and a fallback dimension-decrement mechanism to guarantee reduction even when rounding constraints interfere.

### [Grid Deformation](5372-grid-deformation.md)
Grid Deformation is a preprocessing technique applied prior to data reduction in volume rendering compression. It transforms the input data grid according to user-defined regions to better preserve salient features during subsequent downsampling. It is an orthogonal approach to hierarchical models like wavelets, reporting improved visual results compared to pure downsampling and wavelet-based methods.

### [Optimization-based Downsampling](5825-optimization-based-downsampling.md)
Optimization-based downsampling reduces the resolution of microflake volumes by iteratively optimizing scattering parameters so that the rendered appearance of the lower-resolution level of detail matches a high-resolution reference. It solves the problem of geometric aliasing and shadowing artifacts in aggregated microgeometry, distinguishing itself from naive averaging or simple SGGX approximation by explicitly accounting for shadowing effects through image-space error minimization.

### [Pre-processing for Half Resolution](6436-pre-processing-for-half-resolution.md)
This method constructs a half-resolution volumetric representation from a full-resolution input to reduce data size while preserving visual quality and avoiding the information loss associated with direct subsampling. It operates directly in the spatial domain using a specific odd-index subsampling pattern and local median replacement to fill transparent regions.

### Max-Pooling Layer
A layer that applies a max filter to non-overlapping sub-regions of the input to downsample the input and reduce parameters in subsequent encoder layers [S4, unofficial]. Contains no trainable parameters [S4, unofficial]. Filter Size: 2x2 [S4, unofficial]. Stride: 2 [S4, unofficial]. Applied in layers with input channels 64, 32, and 8 [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Efficient for building multi-resolution hierarchies on GPU; max-pooling is hardware-accelerated; point cloud voxelization is robust to noise [S2, unofficial][S4, unofficial][S3, unofficial].
*   **Limitations:** Max pooling loses high-frequency detail; the specific GPU kernel [S2, unofficial] has complex boundary handling and potential precision issues with the 0.5 averaging weights; point cloud method uses more memory by processing all points in parallel [S3, unofficial].
*   **When to use:** When building VCT cascades [S2, unofficial], reducing feature map size in CNNs [S4, unofficial], or simplifying point clouds for collision detection [S3, unofficial].
*   **When not to use:** When preserving high-frequency details is critical without post-processing; when memory is extremely constrained and the grid is sparse (use sparse voxel octrees instead) [G].

## Background
The specific "Cascade Repacking" implementation is associated with Cascaded Voxel Cone Tracing (CVCT) [S2, unofficial]. The point cloud voxelization method is part of volumetric structures for robotics and SDF queries [S3, unofficial]. The max-pooling approach is standard in deep learning architectures for compressed volume rendering [S4, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S3] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
[S4] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
