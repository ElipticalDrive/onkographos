# Point Cloud Density Evaluation

## Overview
Point Cloud Density Evaluation updates the intensity values of a set of 3D points by sampling from an underlying volume representation, serving as a supervision or density estimation step in point-based volume rendering pipelines [S1, unofficial]. It solves the problem of transferring scalar field information from a continuous volume grid to discrete point locations, enabling differentiable training of point clouds against volumetric targets [S1, unofficial]. Unlike sibling methods that rasterize splats or manage hierarchical data structures, this method specifically handles the interpolation and optional density pooling of intensity values for existing point coordinates [S1, unofficial].

## How it works
**Inputs:**
*   `volume`: 3D or 4D tensor representing the scalar field [S1, unofficial].
*   `points`: Point coordinates in normalized `[0,1]` space, shape `[N, 3]` [S1, unofficial].
*   `scale`: Optional scale parameters for each point, shape `[N, 3]` or `[N]` [S1, unofficial].
*   `min_val`, `max_val`: Optional bounds for the volume; if `None`, computed from the volume data [S1, unofficial].
*   `normalize`: Boolean flag to normalize intensities to `[0,1]` (default `False`) [S1, unofficial].
*   `enable_footprint_pooling`: Boolean flag to aggregate values from neighboring voxels based on point scale [S1, unofficial].

**Outputs:**
*   `intensities`: Tensor of sampled intensity values for each point, shape `[N]` [S1, unofficial].
*   `volume_min`: The minimum value in the volume [S1, unofficial].
*   `volume_max`: The maximum value in the volume [S1, unofficial].

**Data Structures:**
*   `volume_f`: The volume tensor converted to `work_dtype` (typically `torch.float32`) [S1, unofficial].

**Steps:**
1.  **Handle Empty Input:** If `points.numel() == 0`, return an empty tensor for intensities, and `volume.min()`, `volume.max()` [S1, unofficial].
2.  **Type Conversion:** Convert `volume` and `points` to `work_dtype` [S1, unofficial].
3.  **Determine Bounds:** Use `min_val`/`max_val` if provided; otherwise compute `volume_min = volume_f.min()` and `volume_max = volume_f.max()` [S1, unofficial].
4.  **Base Sampling:** Call `sample_intensities_from_volume` with `enable_footprint_pooling=False` to obtain `base_samples` (raw interpolated values) [S1, unofficial]. This uses bilinear interpolation with `align_corners=True` on the normalized point coordinates [S1, unofficial].
5.  **Footprint Pooling (Optional):** If `enable_footprint_pooling` is `True` and `scale` is valid, compute coverage mask indices, convert points to voxel coordinates, compute voxel extent, determine bounding box indices, iterate over each point to extract the voxel region, compute the mean, and update the sample value [S1, unofficial].
6.  **Normalization (Optional):** If `normalize` is `True`, compute the denominator as `max(volume_max - volume_min, 1e-8)`, compute `intensities = (intensities - volume_min) / denominator`, and clamp to `[0, 1]` [S1, unofficial].

**Formulas:**
*   Voxel extent: $\text{extent\_xyz} = \frac{\text{scale\_cov}}{\text{voxel\_size\_xyz}.unsqueeze(0)}$, reordered to $\text{extent\_zyx}$ [S1, unofficial].
*   Bounding box indices: $\text{min\_idx} = \lfloor \text{voxel\_coords} - \frac{\text{extent}}{2} \rfloor$, $\text{max\_idx} = \lceil \text{voxel\_coords} + \frac{\text{extent}}{2} \rceil$, clamped to volume dimensions [S1, unofficial].
*   Normalization: $\text{intensities} = \frac{\text{intensities} - \text{volume\_min}}{\max(\text{volume\_max} - \text{volume\_min}, 10^{-8})}$, clamped to $[0, 1]$ [S1, unofficial].

**Parameters:**
*   `points`: Shape `[N, 3]` in normalized `[0,1]` space [S1, unofficial].
*   `scale`: Shape `[N, 3]` or `[N]` [S1, unofficial].
*   `normalize`: Default `False` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Points:** Must return empty tensor to avoid indexing errors [S1, unofficial].
*   **Precision:** Use `float32` for volume and points to ensure stable interpolation [S1, unofficial].
*   **Boundaries:** Clamping voxel indices is critical to prevent out-of-bounds access during footprint pooling [S1, unofficial].
*   **Normalization Denominator:** Use `max(diff, 1e-8)` to prevent division by zero if volume is constant [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def point_cloud_density_evaluation(
    volume: torch.Tensor,
    points: torch.Tensor,
    scale: torch.Tensor = None,
    min_val: float = None,
    max_val: float = None,
    normalize: bool = False,
    enable_footprint_pooling: bool = False,
    volume_shape: tuple = None,
    voxel_size_xyz: torch.Tensor = None
) -> tuple:
    """
    Updates intensity values for points based on their current positions.
    
    Args:
        volume: 3D/4D tensor of volume data.
        points: Point coordinates [N, 3] in [0, 1] space.
        scale: Optional scale [N, 3] or [N].
        min_val, max_val: Optional volume bounds.
        normalize: Whether to normalize output to [0, 1].
        enable_footprint_pooling: Whether to average over voxel neighborhoods.
        volume_shape: Tuple (D, H, W) of the volume.
        voxel_size_xyz: Tensor of voxel sizes.
        
    Returns:
        intensities: [N] tensor of sampled values.
        volume_min: Scalar min value.
        volume_max: Scalar max value.
    """
    # 1. Handle empty points
    if points.numel() == 0:
        return torch.empty(0), volume.min(), volume.max()

    # 2. Convert to work dtype
    work_dtype = torch.float32
    volume_f = volume.to(work_dtype)
    points_f = points.to(work_dtype)

    # 3. Determine bounds
    if min_val is not None and max_val is not None:
        volume_min = min_val
        volume_max = max_val
    else:
        volume_min = volume_f.min()
        volume_max = volume_f.max()

    # 4. Base sampling
    # Prepare grid for F.grid_sample. Points are in [0, 1].
    # F.grid_sample expects coordinates in [-1, 1] if align_corners=False,
    # or maps [0, 1] endpoints to [-1, 1] if align_corners=True.
    grid_coords = 2.0 * points_f - 1.0
    grid_coords = grid_coords.unsqueeze(0).unsqueeze(0) # [1, 1, N, 3]

    # Ensure volume has batch and channel dims for grid_sample
    if volume_f.dim() == 3:
        volume_sample = volume_f.unsqueeze(0).unsqueeze(0) # [1, 1, D, H, W]
    elif volume_f.dim() == 4:
        volume_sample = volume_f.unsqueeze(0) # [1, C, D, H, W]
    else:
        volume_sample = volume_f

    # Sample intensities using bilinear interpolation
    samples = F.grid_sample(
        volume_sample,
        grid_coords,
        mode='bilinear',
        padding_mode='border',
        align_corners=True
    )

    # Flatten to [N]
    intensities = samples.view(points.shape[0], -1).mean(dim=1)

    # 5. Footprint Pooling
    if enable_footprint_pooling and scale is not None and volume_shape is not None and voxel_size_xyz is not None:
        # Convert points to voxel coordinates
        voxel_coords = points_f * torch.tensor(volume_shape, dtype=work_dtype, device=points.device)

        # Compute voxel extent
        scale_cov = scale if scale.dim() == 2 else scale.unsqueeze(1) # [N, 3]
        extent_xyz = scale_cov / voxel_size_xyz.unsqueeze(0) # [N, 3]
        extent_zyx = extent_xyz[:, [2, 1, 0]] # Reorder to Z, Y, X

        # Compute min/max indices
        min_idx = torch.floor(voxel_coords - extent_zyx / 2.0).long()
        max_idx = torch.ceil(voxel_coords + extent_zyx / 2.0).long()

        # Clamp to volume dimensions
        D, H, W = volume_shape
        min_idx[:, 0] = torch.clamp(min_idx[:, 0], 0, D - 1)
        min_idx[:, 1] = torch.clamp(min_idx[:, 1], 0, H - 1)
        min_idx[:, 2] = torch.clamp(min_idx[:, 2], 0, W - 1)
        max_idx[:, 0] = torch.clamp(max_idx[:, 0], 0, D - 1)
        max_idx[:, 1] = torch.clamp(max_idx[:, 1], 0, H - 1)
        max_idx[:, 2] = torch.clamp(max_idx[:, 2], 0, W - 1)

        # Iterate and average
        new_intensities = torch.zeros_like(intensities)
        for i in range(points.shape[0]):
            z0, z1 = min_idx[i, 0], max_idx[i, 0]
            y0, y1 = min_idx[i, 1], max_idx[i, 1]
            x0, x1 = min_idx[i, 2], max_idx[i, 2]

            region = volume_f[z0:z1+1, y0:y1+1, x0:x1+1]
            if region.numel() > 0:
                new_intensities[i] = region.mean()
            else:
                new_intensities[i] = intensities[i]

        intensities = new_intensities

    # 6. Normalization
    if normalize:
        denominator = max(volume_max - volume_min, 1e-8)
        intensities = (intensities - volume_min) / denominator
        intensities = torch.clamp(intensities, 0.0, 1.0)

    return intensities, volume_min, volume_max
```

## Subtopics

### [Chamfer Distance](2446-chamfer-distance.md)
Chamfer Distance (CD) is a geometric metric used to evaluate the reconstruction accuracy of 3D surfaces by measuring the average closest-point distance between two point clouds, typically representing a predicted surface and a ground-truth reference. It serves as a standard benchmark for SDF-based NeRFs and mesh reconstruction pipelines, complementing photometric metrics like PSNR by quantifying geometric fidelity. Unlike sibling metrics such as [Neighbor Counting](2905-neighbor-counting.md) which assess local density, or [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md) which may include normal consistency, CD focuses strictly on the spatial proximity of surface points.

### [Neighbor Counting](2905-neighbor-counting.md)
Neighbor Counting is a fundamental technique in volume data and representation that quantifies the local density or spatial distribution of points within a point cloud or mesh by determining how many other points reside within a specified proximity of a query point. It serves as the primary mechanism for evaluating point cloud density in non-regular volumes, distinguishing itself from continuous density estimators like [Parzen Density Estimation](11796-parzen-density-estimation.md) by providing discrete, integer-based counts rather than smoothed kernel sums. Unlike connectivity-based clustering methods such as [Grid-based DBSCAN](11706-grid-based-dbscan.md) which group points into arbitrary shapes, Neighbor Counting focuses on local neighborhood statistics to inform voxelization, surface reconstruction, or clustering decisions.

### [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md)
BrickDensityRegion Point Splatting maps point cloud data with location and O2 concentration attributes to a 3D voxel grid using normalization, threshold-based splat radii, and a distance-weighted kernel. It differs from sibling methods like [Chamfer Distance](2446-chamfer-distance.md) or [Neighbor Counting](2905-neighbor-counting.md) by accumulating weighted density values into voxels rather than computing discrete metrics or occupancy counts.

### [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md)
Mesh/Point Cloud Evaluation Metrics provides a comprehensive suite of geometric and topological measures to assess the fidelity and quality of reconstructed 3D surfaces against ground truth data. It addresses the problem of objectively quantifying reconstruction errors by combining point-wise proximity (Chamfer Distance), surface alignment (Normal Consistency), and threshold-based correctness (F-score), while also penalizing poor mesh topology (aspect ratios, self-intersections). Unlike [Chamfer Distance](2446-chamfer-distance.md), which only measures average displacement, this method integrates precision/recall trade-offs and mesh regularity checks to provide a holistic quality assessment. It differs from [Cloud-to-Mesh Accuracy Metrics](10560-cloud-to-mesh-accuracy-metrics.md) by evaluating symmetric point-cloud distances rather than orthogonal point-to-facet displacements.

### [Probabilistic Occupancy Policy](5043-probabilistic-occupancy-policy.md)
The Probabilistic Occupancy Policy determines the occupancy status of a point cloud point based on its intensity field value, mapping values near 0.0 to free space and values near 1.0 to occupied space. It distinguishes itself from the Signed Intensity Occupancy Policy by using a symmetric tolerance range around both 0 and 1 rather than a single threshold near zero.

### [Bhattacharyya Distance Coefficient](6998-bhattacharyya-distance-coefficient.md)
The Bhattacharyya Distance Coefficient measures the overlap between two discrete probability distributions, specifically the histograms of voxel values inside and outside a target object. It serves as a density evaluation metric to quantify distribution overlap in volume data. Unlike sibling methods that measure geometric proximity, this method evaluates statistical distribution similarity [G].

### [Plane Standard Deviation (Pσ) Calculation](7880-plane-standard-deviation-p-calculation.md)
The Plane Standard Deviation (Pσ) metric evaluates the geometric accuracy of reconstructed planar surfaces in point clouds, specifically assessing how well a NeRF-generated point cloud aligns with ground-truth LiDAR data. It isolates planar structures by semantic grouping and measures the perpendicular spread of NeRF points relative to the estimated surface normal of the corresponding LiDAR plane. Unlike global metrics, Pσ focuses on local planar fidelity, making it distinct from general Chamfer Distance or neighbor-counting methods by explicitly accounting for surface orientation.

### Poisson Distribution Lookup for Feature Points
Given a uniformly distributed random number, returns the number of feature points in a given cube based on Poisson distribution thresholds [S2, unofficial].
*   If value < 0.0915781944272058, return 1 [S2, unofficial].
*   If value < 0.238103305510735, return 2 [S2, unofficial].
*   If value < 0.433470120288774, return 3 [S2, unofficial].
*   If value < 0.628836935299644, return 4 [S2, unofficial].
*   If value < 0.785130387122075, return 5 [S2, unofficial].
*   If value < 0.889326021747972, return 6 [S2, unofficial].
*   If value < 0.948866384324819, return 7 [S2, unofficial].
*   If value < 0.978636565613243, return 8 [S2, unofficial].

### Point Cloud Size Ablation
An evaluation of the trade-off between point cloud density and performance metrics (training time, rendering speed, model size, and PSNR) [S3]. The chosen number of points for experiments is 45k, balancing quality and performance [S3].
*   **15k points:** Train time 16 min, Render 35 fps, Model size 37 MB, PSNR 32.2 dB [S3].
*   **30k points:** Train time 20 min, Render 32 fps, Model size 73 MB, PSNR 33.6 dB [S3].
*   **45k points:** Train time 30 min, Render 25 fps, Model size 110 MB, PSNR 34.6 dB [S3].
*   **60k points:** Train time 41 min, Render 16 fps, Model size 146 MB, PSNR 34.8 dB [S3].
*   **100k points:** Train time 76 min, Render 10 fps, Model size 244 MB, PSNR 34.2 dB [S3].

### Stretched Uniform Distributions
A technique to generate weighted point densities in 2D by generating a uniform point density and pulling points radially into the center based on a reconstruction function [S4, unofficial]. Let $h(r)$ be the point density. The expected number of points in an infinitesimal ring at radius $r$ with width $dr$ is $h(r) 2\pi r dr$. For a uniform distribution with density $k$ in 'flat land' with radius $p$, the expected number is $k 2\pi p dp$. Equating densities: $k 2\pi p dp = h(r) 2\pi r dr$. Integrating both sides yields $\int_0^p k p dp = \int_0^r h(r) r dr$, leading to $k \frac{p^2}{2} = \int_0^r h(r) r dr$. Solving for $p$ leads to the mapping function. The inverse of this function is needed to map from $p$ to $r$. If the reconstruction function is proportional to a gaussian, this can be solved analytically [S4, unofficial].

### Instance Segmentation from Dynamic Density
Derives 3D instance count, centroids, and predictions from the dynamic density field without requiring labeled data, using geometric clustering [S5].
*   Sample dynamic density values $\sigma_d$ from the scene representation [S5].
*   Project density values that exceed a given threshold onto a discretized ground plane [S5].
*   Apply connected component labeling to the projected points to identify distinct instances [S5].
*   Apply k-means clustering to the labeled points to obtain 3D centroids for each instance [S5].
*   Volume render the instance predictions using the computed centroids and assignments [S5].

## Operating characteristics
*   **Time Complexity:** Base sampling is $O(N)$ where $N$ is the number of points, assuming $O(1)$ grid sampling [S1, unofficial]. Footprint pooling adds complexity proportional to the volume of the footprint region per point, potentially $O(N \cdot R^3)$ where $R$ is the radius in voxels [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing point intensities [S1, unofficial]. Volume storage is $O(D \cdot H \cdot W)$ [S1, unofficial].
*   **Measured Performance:** Based on Point Cloud Size Ablation [S3], increasing point count from 15k to 100k increases training time from 16 min to 76 min and decreases render FPS from 35 to 10, while PSNR improves from 32.2 dB to a peak of 34.8 dB at 60k points [S3].
*   **Strengths:** Differentiable, allows direct supervision of point clouds from volume data, supports adaptive density via footprint pooling [S1, unofficial].
*   **Limitations:** Performance degrades with high point counts and large footprints [S1, unofficial]. Requires careful normalization and boundary handling [S1, unofficial].
*   **When to use:** When training point-based representations against volumetric ground truth or when evaluating point cloud density against a volume field [S1, unofficial].
*   **When not to use:** When only geometric metrics are needed without intensity supervision, or when memory is too constrained for large volume grids [S1, unofficial].

## Background
Originating from the 3D Gaussian Splatting ecosystem and its acceleration variants, this method is used to supervise point clouds (Gaussians) using volume data [S1, unofficial]. It builds on the original 3D Gaussian Splatting work by Kerbl et al. [S1, unofficial]. The method is implemented in repositories such as `3dgs_accel` [S1, unofficial].

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S3] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S4] Multivariate volume rendering, https://www.osti.gov/biblio/495676, fetched 2026-10-26 (unofficial)
[S5] SUDS: Scalable Urban Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.01191, fetched 2026-10-26
