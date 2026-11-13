# Point Cloud Cropping and Downsampling

Also called: Crop and Downsample Point Cloud, Voxel Grid Filtering, VoxelGrid Filter

## Overview
This method reduces the density of a point cloud by first removing points outside a specified spatial volume (cropping) and then subsampling the remaining points to a target resolution (downsampling) [S1, unofficial]. It solves the problem of excessive point cloud size and noise in unprojected data, which hinders downstream processing like registration or rendering [S2, unofficial]. It sets itself apart from sibling methods like [Hard Voxelization](5345-hard-voxelization.md) by offering a choice between voxel-based averaging and uniform random sampling, and from [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md) by relying on global spatial binning or simple random selection rather than local geometric consistency metrics.

## How it works
**Inputs:**
- Point cloud $P$ with $N$ points, where each point $p_i \in \mathbb{R}^3$ has coordinates $(x, y, z)$.
- Cropping volume $V$, defined as an axis-aligned bounding box (AABB) with minimum corner $(x_{min}, y_{min}, z_{min})$ and maximum corner $(x_{max}, y_{max}, z_{max})$.
- Downsampling method $M \in \{\text{'voxel'}, \text{'uniform'}\}$. Default: 'voxel' [S1, unofficial].
- Voxel size $s$ (leaf size) in meters. Default: $0.01$ [S1, unofficial] or $(0.025, 0.025, 0.025)$ [S2, unofficial].
- Maximum point number $N_{max}$ for uniform downsampling. Value not specified in code snippet [S1, unofficial].

**Outputs:**
- Downsampled point cloud $P'$ with $N' \le N$ points.

**Data Structures:**
- The point cloud is typically stored as a list or array of 3D coordinates.
- For voxel downsampling, a hash map or grid structure maps voxel indices $(i, j, k)$ to a list of points or an accumulated centroid.

**Steps:**

1. **Input Validation and Cleaning:**
   Filter the input point cloud $P$ to remove any points containing NaN (Not a Number) values in their coordinates [S2, unofficial]. Let the cleaned cloud be $P_{clean}$.

2. **Cropping:**
   Iterate through each point $p \in P_{clean}$. Keep $p$ if and only if:
   $$
   x_{min} \le p.x \le x_{max} \quad \land \quad y_{min} \le p.y \le y_{max} \quad \land \quad z_{min} \le p.z \le z_{max}
   $$
   Let the cropped cloud be $P_{crop}$.

3. **Downsampling:**
   If $M = \text{'voxel'}$:
   a. Compute the voxel grid dimensions based on the crop volume bounds and voxel size $s$.
   b. For each point $p \in P_{crop}$, compute its voxel index $(i, j, k)$:
      $$
      i = \lfloor \frac{p.x - x_{min}}{s} \rfloor, \quad j = \lfloor \frac{p.y - y_{min}}{s} \rfloor, \quad k = \lfloor \frac{p.z - z_{min}}{s} \rfloor
      $$
   c. Accumulate points into voxels. For each voxel containing points, compute the centroid:
      $$
      c_{ijk} = \frac{1}{|V_{ijk}|} \sum_{p \in V_{ijk}} p
      $$
      where $V_{ijk}$ is the set of points in voxel $(i, j, k)$.
   d. The output $P'$ consists of all centroids $c_{ijk}$.

   If $M = \text{'uniform'}$:
   a. If the number of points in $P_{crop}$ exceeds $N_{max}$, randomly select $N_{max}$ points from $P_{crop}$ without replacement.
   b. If the number of points is $\le N_{max}$, keep all points.
   c. The output $P'$ is the selected subset.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def crop_and_downsample(points, crop_volume, method='voxel', voxel_size=0.01, max_points=None):
    """
    points: list of tuples (x, y, z)
    crop_volume: tuple (x_min, y_min, z_min, x_max, y_max, z_max)
    method: 'voxel' or 'uniform'
    voxel_size: float, size of the voxel edge
    max_points: int, max points for uniform downsampling
    """
    x_min, y_min, z_min, x_max, y_max, z_max = crop_volume
    
    # Step 1: Filter NaN values
    cleaned_points = []
    for p in points:
        if not (math.isnan(p[0]) or math.isnan(p[1]) or math.isnan(p[2])):
            cleaned_points.append(p)
    
    # Step 2: Crop
    cropped_points = []
    for p in cleaned_points:
        if x_min <= p[0] <= x_max and y_min <= p[1] <= y_max and z_min <= p[2] <= z_max:
            cropped_points.append(p)
            
    # Step 3: Downsample
    if method == 'voxel':
        voxel_map = {}
        for p in cropped_points:
            # Compute voxel index
            i = int(math.floor((p[0] - x_min) / voxel_size))
            j = int(math.floor((p[1] - y_min) / voxel_size))
            k = int(math.floor((p[2] - z_min) / voxel_size))
            key = (i, j, k)
            
            if key not in voxel_map:
                voxel_map[key] = []
            voxel_map[key].append(p)
            
        downsampled_points = []
        for key, pts in voxel_map.items():
            # Compute centroid
            sum_x = sum(p[0] for p in pts)
            sum_y = sum(p[1] for p in pts)
            sum_z = sum(p[2] for p in pts)
            count = len(pts)
            downsampled_points.append((sum_x / count, sum_y / count, sum_z / count))
            
    elif method == 'uniform':
        if max_points is not None and len(cropped_points) > max_points:
            import random
            downsampled_points = random.sample(cropped_points, max_points)
        else:
            downsampled_points = cropped_points
    else:
        raise ValueError("Unknown downsampling method")
        
    return downsampled_points
```

**Parameters:**
- `voxel_size`: Voxel size for downsampling. Default: $0.01$ [S1, unofficial]. Specific implementations may use $(0.025, 0.025, 0.025)$ [S2, unofficial].
- `max_points`: Maximum number of points before uniform downsampling is triggered. Value not specified in code snippet [S1, unofficial]. In some registration contexts, this may default to $4 \times \text{MAX\_POINT\_NUMBER}$ but is noted as not used in current implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **NaN Values:** Input clouds may contain NaN coordinates. These must be filtered out before cropping or downsampling to prevent errors in centroid calculation or random sampling [S2, unofficial].
- **Empty Voxels:** If a voxel contains no points, it is ignored.
- **Boundary Conditions:** Points exactly on the boundary of the crop volume are included if the condition is inclusive ($\le$).
- **Precision:** Floating-point precision in voxel index calculation can lead to jitter if not handled consistently. Using `floor` ensures deterministic binning.

## Subtopics

### [Farthest Point Sampling (FPS)](7182-farthest-point-sampling-fps.md)
Farthest Point Sampling (FPS) is a deterministic (after initial random seed) algorithm for selecting a subset of points from a point cloud such that the selected points are maximally spread out in Euclidean space, ensuring maximum spatial coverage. It solves the problem of uniform downsampling where simple random sampling may leave gaps or cluster points, and grid-based methods may lose fine geometric details. It sets itself apart from sibling methods like [Adaptive Point Cloud Downsampling based on Geometry and Texture Complexity](10729-adaptive-point-cloud-downsampling-based-on-geometry-and-text.md) by relying purely on geometric distance metrics rather than local feature complexity or texture density.

### [Point Cloud Filtering and Noise Removal](8464-point-cloud-filtering-and-noise-removal.md)
Removes statistical outliers from a point cloud to clean noise and artifacts, specifically targeting clear floaters in unprojected point clouds. It differs from sibling methods like Farthest Point Sampling, which focuses on uniform spatial distribution, by preserving the original density of inliers while discarding points that deviate significantly from local geometric consistency.

### [Adaptive Point Cloud Downsampling based on Geometry and Texture Complexity](10729-adaptive-point-cloud-downsampling-based-on-geometry-and-text.md)
This method reduces the density of a point cloud by selecting a subset of points based on local geometric curvature and color texture complexity, ensuring that visually or structurally significant regions are preserved during downsampling. It differs from uniform voxel downsampling by using probabilistic sampling weighted by complexity metrics rather than simple spatial binning, and from uniform random downsampling by biasing selection toward high-complexity areas. It builds on [Point Cloud Cropping and Downsampling](7057-point-cloud-cropping-and-downsampling.md).

### Voxel Downsampling Registration
Performs ICP registration between source and ground truth target point clouds using voxel downsampling, with an initial transformation [S1, unofficial].
1. If verbose: print voxel_size and threshold, set Open3D verbosity to Debug.
2. Downsample source: `crop_and_downsample(source, crop_volume, 'voxel', voxel_size, trans=init_trans)`.
3. Downsample target: `crop_and_downsample(gt_target, crop_volume, 'voxel', voxel_size)`.
4. Perform ICP: `registration_icp(s, t, threshold, np.identity(4), TransformationEstimationPointToPoint(True), ICPConvergenceCriteria(1e-6, max_itr))`.
5. Update transformation: `reg.transformation = matmul(reg.transformation, init_trans)`.
6. Return registration result.
Parameters: `voxel_size` is required [S1, unofficial]. ICP convergence tolerance is $10^{-6}$, max iterations `max_itr` [S1, unofficial].

### Uniform Registration
Performs ICP registration between source and ground truth target point clouds using uniform downsampling, with an initial transformation [S1, unofficial].
1. If verbose: print threshold, set Open3D verbosity to Debug.
2. Downsample source: `crop_and_downsample(source, crop_volume, 'uniform', trans=init_trans)`.
3. Downsample target: `crop_and_downsample(gt_target, crop_volume, 'uniform')`.
4. Perform ICP: `registration_icp(s, t, threshold, np.identity(4), TransformationEstimationPointToPoint(True), ICPConvergenceCriteria(1e-6, max_itr))`.
5. Update transformation: `reg.transformation = matmul(reg.transformation, init_trans)`.
6. Return registration result.
Parameters: `max_size` defaults to $4 \times \text{MAX\_POINT\_NUMBER}$ but is not used in current implementation [S1, unofficial]. ICP convergence tolerance is $10^{-6}$, max iterations `max_itr` [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Cropping is $O(N)$ where $N$ is the number of points. Voxel downsampling is $O(N)$ on average with hash map insertion, but depends on the number of occupied voxels. Uniform downsampling is $O(N)$ for filtering and $O(K)$ for sampling where $K$ is the output size.
- **Memory Complexity:** $O(N)$ for storing the input and intermediate cropped points. Voxel downsampling requires additional memory for the voxel grid, proportional to the number of occupied voxels.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of points. Voxel size affects the number of voxels and thus the memory overhead and output size.
- **Strengths:** Simple to implement, effective for reducing point cloud density, handles noise via voxel averaging.
- **Limitations:** Voxel downsampling may lose fine geometric details if the voxel size is too large. Uniform downsampling may result in non-uniform spatial distribution.
- **When to use:** When a quick reduction in point cloud size is needed for registration or visualization.
- **When not to use:** When preserving fine geometric details is critical, or when the point cloud has significant anisotropic density.

## Background
Origin: This method is commonly implemented in point cloud processing libraries such as Open3D [S1, unofficial][S2, unofficial]. It is a standard preprocessing step in 3D computer vision pipelines, particularly for LiDAR data and 3D reconstruction.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] zAle711/Point-Cloud-Voxelization-Node, https://github.com/zAle711/Point-Cloud-Voxelization-Node, fetched 2026-10-27 (unofficial)
