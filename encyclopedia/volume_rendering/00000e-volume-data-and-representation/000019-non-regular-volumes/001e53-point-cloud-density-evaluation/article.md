# Point Cloud Density Evaluation


## Overview
Point Cloud Density Evaluation is a method for sampling scalar field values from a volumetric representation onto a set of 3D points, typically used to supervise or evaluate point-based volume representations (such as 3D Gaussians) against a ground-truth or auxiliary volume. It solves the problem of transferring continuous volumetric information to discrete point locations, enabling density-aware supervision and error calculation. It differs from sibling methods like [Chamfer Distance](2446-chamfer-distance.md) by operating on intensity fields rather than geometric proximity, and from [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md) by sampling existing volumes rather than constructing new density fields from points.

## How it works
**Inputs:**
- `points`: Tensor of point coordinates, shape $[N, 3]$, in normalized $[0, 1]$ space [S1, unofficial].
- `volume`: 3D or 5D tensor representing the scalar field (e.g., density or intensity) [S1, unofficial].
- `scale`: Optional scale parameters for each point, shape $[N, 3]$ or $[N]$ [S1, unofficial].
- `min_val`, `max_val`: Optional bounds for normalization; if `None`, computed from the volume data [S1, unofficial].
- `normalize`: Boolean flag to normalize output intensities to $[0, 1]$ (default `False`) [S1, unofficial].
- `enable_footprint_pooling`: Boolean flag to enable spatial aggregation around points (default `False` in base call) [S1, unofficial].

**Outputs:**
- `intensities`: Tensor of sampled intensity values, shape $[N]$ [S1, unofficial].
- `volume_min`: Minimum value in the volume (for normalization reference) [S1, unofficial].
- `volume_max`: Maximum value in the volume (for normalization reference) [S1, unofficial].

**Data Structures:**
- The volume is treated as a grid. When `enable_footprint_pooling` is active, voxel coordinates are computed.
- Voxel extent is calculated as `extent_xyz = scale_cov / voxel_size_xyz.unsqueeze(0)`, reordered to `extent_zyx` [S1, unofficial].
- Bounding box indices in voxel space are computed using `floor` and `ceil` on the voxel coordinates, clamped to volume dimensions [S1, unofficial].

**Steps:**
1. **Empty Check:** If `points.numel() == 0`, return an empty tensor for intensities, and the volume's min/max values [S1, unofficial].
2. **Type Conversion:** Convert `volume` and `points` to `work_dtype` (typically `torch.float32`) [S1, unofficial].
3. **Determine Bounds:** Set `volume_min` and `volume_max`. If `min_val`/`max_val` are provided, use them; otherwise, compute `volume_f.min()` and `volume_f.max()` [S1, unofficial].
4. **Base Sampling:** Call `sample_intensities_from_volume` with `enable_footprint_pooling=False` to obtain `base_samples` (raw interpolated values) [S1, unofficial]. This typically involves bilinear/trilinear interpolation via `F.grid_sample` with `align_corners=True` [S1, unofficial].
5. **Footprint Pooling (Optional):** If `enable_footprint_pooling` is `True` and `scale` is valid:
   - Compute coverage mask indices.
   - Convert points to voxel coordinates using the scale and voxel size [S1, unofficial].
   - Determine the voxel bounding box `[z0:z1+1, y0:y1+1, x0:x1+1]` for each point [S1, unofficial].
   - Iterate over each point, extract the voxel region, compute the mean of the values in that region, and update the sample value [S1, unofficial].
6. **Normalization (Optional):** If `normalize` is `True`:
   - Compute denominator: `denominator = max(volume_max - volume_min, 1e-8)` [S1, unofficial].
   - Compute intensities: `intensities = (intensities - volume_min) / denominator` [S1, unofficial].
   - Clamp values to $[0, 1]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def point_cloud_density_evaluation(points, volume, scale=None, min_val=None, max_val=None, normalize=False, enable_footprint_pooling=False):
    """
    Samples intensity values from a volume for a set of points.
    
    Args:
        points: Tensor of shape [N, 3], normalized [0, 1].
        volume: Tensor representing the volume.
        scale: Optional tensor of shape [N, 3] or [N].
        min_val, max_val: Optional bounds.
        normalize: Whether to normalize output to [0, 1].
        enable_footprint_pooling: Whether to aggregate neighbors.
        
    Returns:
        intensities: Tensor of shape [N].
        volume_min: Scalar.
        volume_max: Scalar.
    """
    # Step 1: Handle empty points
    if points.numel() == 0:
        return torch.empty(0), volume.min(), volume.max()

    # Step 2: Convert to work dtype
    work_dtype = torch.float32
    volume_f = volume.to(work_dtype)
    points_f = points.to(work_dtype)
    
    # Step 3: Determine volume bounds
    if min_val is not None:
        volume_min = min_val
    else:
        volume_min = volume_f.min()
        
    if max_val is not None:
        volume_max = max_val
    else:
        volume_max = volume_f.max()

    # Step 4: Base sampling (simplified grid_sample logic)
    # Note: In practice, this requires mapping points [0,1] to grid coordinates [-1, 1]
    # and reshaping for F.grid_sample.
    # Assuming a helper `sample_intensities_from_volume` exists as per source logic.
    base_samples = sample_intensities_from_volume(points_f, volume_f, enable_footprint_pooling=False)
    
    intensities = base_samples

    # Step 5: Footprint pooling (if enabled)
    if enable_footprint_pooling and scale is not None:
        # Compute voxel coordinates
        # voxel_size_xyz is assumed to be derived from volume shape
        # extent_xyz = scale_cov / voxel_size_xyz.unsqueeze(0)
        # extent_zyx = extent_xyz[:, [2, 1, 0]]
        
        # Convert points to voxel indices
        # voxel_coords = points_f / voxel_size_xyz
        # min_idx = floor(voxel_coords - extent_zyx)
        # max_idx = ceil(voxel_coords + extent_zyx)
        # Clamp min_idx and max_idx to volume dimensions
        
        # Iterate and aggregate
        # for i in range(N):
        #     region = volume_f[min_idx[i]:max_idx[i]+1]
        #     intensities[i] = region.mean()
        pass # Implementation detail omitted as source describes iteration logic without full grid indexing code

    # Step 6: Normalization
    if normalize:
        denominator = max(volume_max - volume_min, 1e-8)
        intensities = (intensities - volume_min) / denominator
        intensities = torch.clamp(intensities, 0.0, 1.0)

    return intensities, volume_min, volume_max

def sample_intensities_from_volume(points, volume, enable_footprint_pooling):
    """
    Helper to perform bilinear interpolation.
    """
    # Map points from [0, 1] to [-1, 1] for grid_sample
    grid_coords = 2.0 * points - 1.0
    grid_coords = grid_coords.unsqueeze(0).unsqueeze(0) # Shape [1, 1, N, 3]
    
    # volume should be [1, C, D, H, W] for F.grid_sample
    # Assuming volume is [D, H, W] or [1, 1, D, H, W]
    if volume.dim() == 3:
        volume_5d = volume.unsqueeze(0).unsqueeze(0)
    elif volume.dim() == 4:
        volume_5d = volume.unsqueeze(0)
    else:
        volume_5d = volume
        
    samples = F.grid_sample(
        volume_5d,
        grid_coords,
        mode="bilinear",
        padding_mode="zeros",
        align_corners=True
    )
    return samples.view(points.shape[0])
```

**Parameters:**
- `points`: Shape $[N, 3]$, normalized $[0, 1]$ [S1, unofficial].
- `scale`: Optional, shape $[N, 3]$ or $[N]$ [S1, unofficial].
- `normalize`: Default `False` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Input:** If `points` is empty, the function returns immediately with empty tensors [S1, unofficial].
- **Precision:** Use `float32` for work dtype to ensure stability during interpolation [S1, unofficial].
- **Normalization Denominator:** Use `max(volume_max - volume_min, 1e-8)` to avoid division by zero [S1, unofficial].

## Subtopics

### [Chamfer Distance](2446-chamfer-distance.md)
Chamfer Distance (CD) is a symmetric metric that evaluates the geometric fidelity between two point clouds by computing the average nearest-neighbor distance in both directions. It solves the problem of quantifying reconstruction accuracy for 3D surfaces and point sets, serving as a primary loss function in differentiable rendering and mesh reconstruction. Unlike sibling metrics such as [Neighbor Counting](2905-neighbor-counting.md) which assesses local density, or [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md) which evaluates volumetric occupancy, CD measures global surface alignment. It differs from [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md) by providing a continuous differentiable loss rather than a binary pass/fail threshold, and from [Probabilistic Occupancy Policy](5043-probabilistic-occupancy-policy.md) by operating on deterministic point coordinates rather than probability distributions.

### [Neighbor Counting](2905-neighbor-counting.md)
Neighbor Counting is a fundamental technique in Point Cloud Density Evaluation that estimates the local density of points by counting the number of neighbors within a specified radius for each point in a dataset. It serves as a core primitive for assessing spatial distribution, identifying outliers, and initializing density-based clustering or rendering parameters. Unlike global metrics, it provides a per-point local density estimate, distinguishing it from sibling methods like Chamfer Distance which measure set-to-set error, or BrickDensityRegion Point Splatting which aggregates density into voxel bricks [3956-brickdensityregion-point-splatting.md].

### [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md)
BrickDensityRegion Point Splatting is a method for converting sparse point cloud data into a dense volumetric density representation within a hierarchical "brick" structure. It solves the problem of efficiently populating voxel grids with density values derived from point attributes (specifically O2 concentration) by projecting points onto the grid using a distance-weighted kernel. It differs from sibling methods like [Chamfer Distance](2446-chamfer-distance.md) or [Neighbor Counting](2905-neighbor-counting.md) by explicitly generating a continuous density field suitable for volume rendering, rather than computing discrete metrics or occupancy counts.

### [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md)
This method computes a comprehensive suite of geometric fidelity metrics to evaluate the quality of reconstructed 3D meshes or point clouds against ground truth data. It addresses the problem of quantifying both surface proximity and geometric regularity, providing scalar scores for Chamfer Distance (CD), Normal Consistency (NC), F1 score, Edge Chamfer Distance (ECD), and mesh shape quality indicators. It differs from standard mesh evaluation by explicitly calculating edge-specific metrics (ECD/EF1) and shape regularity (aspect ratio, angle regularity) alongside standard distance-based metrics, offering a more granular assessment of mesh topology and triangle quality.

### [Probabilistic Occupancy Policy](5043-probabilistic-occupancy-policy.md)
The Probabilistic Occupancy Policy determines the occupancy status of a point cloud point based on its intensity field value, using a probabilistic interpretation where values near 0.0 indicate free space and values near 1.0 indicate occupied space. It sets itself apart from the Signed Intensity Occupancy Policy by using a symmetric tolerance range around both 0 and 1, rather than a single threshold near zero.

### [Bhattacharyya Distance Coefficient](6998-bhattacharyya-distance-coefficient.md)
The Bhattacharyya Distance Coefficient measures the overlap between two discrete probability distributions, specifically the histograms of voxel values inside and outside a target object in non-regular volumes. It serves as a density evaluation metric that quantifies how well the internal and external intensity distributions align, distinguishing itself from sibling methods like [Chamfer Distance](2446-chamfer-distance.md) by focusing on distributional similarity rather than geometric point-to-point proximity.

### Poisson Distribution Lookup for Feature Points
Given a uniformly distributed random number, returns the number of feature points in a given cube based on Poisson distribution thresholds [S2, unofficial]. The thresholds are: if value < 0.0915781944272058, return 1; if value < 0.238103305510735, return 2; if value < 0.433470120288774, return 3; if value < 0.628836935299644, return 4; if value < 0.785130387122075, return 5; if value < 0.889326021747972, return 6; if value < 0.948866384324819, return 7; if value < 0.978636565613243, return 8 [S2, unofficial].

### Point Cloud Size Ablation
An evaluation of the trade-off between point cloud density and performance metrics (training time, rendering speed, model size, and PSNR) [S3]. The chosen number of points for experiments is 45k, balancing quality and performance [S3]. Performance metrics are:
- 15k points: Train time 16 min, Render 35 fps, Model size 37 MB, PSNR 32.2 dB.
- 30k points: Train time 20 min, Render 32 fps, Model size 73 MB, PSNR 33.6 dB.
- 45k points: Train time 30 min, Render 25 fps, Model size 110 MB, PSNR 34.6 dB.
- 60k points: Train time 41 min, Render 16 fps, Model size 146 MB, PSNR 34.8 dB.
- 100k points: Train time 76 min, Render 10 fps, Model size 244 MB, PSNR 34.2 dB [S3].

### Plane Standard Deviation (Pσ) Calculation
To evaluate plane reconstruction, the standard deviation of NeRF predicted points along the estimated normals from the LiDAR point cloud is calculated [S4]. For a given semantic group representing a single plane, the filtered LiDAR point cloud and the corresponding NeRF point cloud are extracted [S4]. Both point clouds are split into patches of size $3 \times 3$ square meters [S4]. For each pair of LiDAR and NeRF patches, the RANSAC algorithm is used to estimate the normal of the LiDAR patch, and the standard deviation of the patch along this estimated normal is calculated [S4].

### Stretched Uniform Distributions
A technique to generate weighted point densities in 2D by generating a uniform point density and pulling points radially into the center based on a reconstruction function [S5, unofficial]. Let $h(r)$ be the point density. The expected number of points in an infinitesimal ring at radius $r$ with width $dr$ is $h(r) 2\pi r dr$. For a uniform distribution with density $k$ in 'flat land' with radius $p$, the expected number is $k 2\pi p dp$. Equating densities: $k 2\pi p dp = h(r) 2\pi r dr$ [S5, unofficial]. Integrating both sides yields $\int_0^p k p dp = \int_0^r h(r) r dr$, leading to $k \frac{p^2}{2} = \int_0^r h(r) r dr$. Solving for $p$ leads to the mapping function, and the inverse of this function is needed to map from $p$ to $r$ [S5, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the interpolation step, which is $O(N)$ for $N$ points, assuming constant-time grid lookup. Footprint pooling adds a factor proportional to the number of voxels in the footprint region.
- **Memory Complexity:** $O(N)$ for storing point coordinates and sampled intensities. The volume itself is $O(D \times H \times W)$.
- **Measured Performance:** Based on [S3], increasing point cloud size from 15k to 100k increases training time from 16 min to 76 min and model size from 37 MB to 244 MB, while rendering FPS drops from 35 to 10. PSNR improves from 32.2 dB to a peak of 34.8 dB at 60k points, then slightly decreases to 34.2 dB at 100k.
- **Strengths:** Efficiently transfers volumetric information to points; supports normalization and footprint pooling for robust supervision.
- **Limitations:** Performance degrades with very high point counts; relies on the underlying volume resolution for accuracy.

## Background
The method is part of the 3D Gaussian Splatting ecosystem, building on the original work by Kerbl et al. [S1, unofficial]. It is used in repositories such as `ValentinKraft/vgs` for volume-only training and evaluation [S1, unofficial]. The specific ablation studies on point cloud size are from "Differentiable Point-Based Radiance Fields for Efficient View Synthesis" [S3].

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S3] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S4] PlaNeRF: SVD Unsupervised 3D Plane Regularization for NeRF Large-Scale Urban Scene Reconstruction, https://doi.org/10.1109/3dv62453.2024.00114, fetched 2026-11-04
[S5] Multivariate volume rendering, https://www.osti.gov/biblio/495676, fetched 2026-10-26 (unofficial)
