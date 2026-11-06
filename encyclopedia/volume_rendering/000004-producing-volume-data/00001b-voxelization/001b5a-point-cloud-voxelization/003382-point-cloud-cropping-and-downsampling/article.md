# Point Cloud Cropping and Downsampling

Also called: Crop and Downsample Point Cloud

## Overview
This method prepares point clouds for voxelization or registration by removing points outside a specified volume and reducing point density to a manageable size [S1, unofficial]. It solves the problem of excessive memory usage and computational cost in downstream tasks like ICP registration or sparse voxel grid construction by ensuring the input point cloud fits within hardware limits and spatial resolution constraints [S1, unofficial]. It is distinguished from sibling methods like [Hard Voxelization](5345-hard-voxelization.md) by operating on the raw point set prior to voxel assignment, rather than during the voxelization process itself.

## How it works
**Inputs:**
- `point_cloud`: A point cloud object containing positions (and optionally attributes like normals or colors).
- `crop_volume`: A bounding volume (e.g., axis-aligned bounding box or sphere) defining the region of interest.
- `down_sample_method`: String, either `'voxel'` or `'uniform'`. Default: `'voxel'` [S1, unofficial].
- `voxel_size`: Float, the side length of the voxel grid used for downsampling. Default: `0.01` [S1, unofficial].
- `max_size`: Integer, maximum number of points allowed. Used only for uniform downsampling. Default: `4 * MAX_POINT_NUMBER` (where `MAX_POINT_NUMBER` is not explicitly defined in the source snippet) [S1, unofficial].
- `init_trans`: Transformation matrix (optional), applied to the source point cloud before downsampling in registration contexts [S1, unofficial].

**Outputs:**
- A cropped and downsampled point cloud.

**Data Structures:**
- The point cloud is typically represented as an array of points, where each point has 3D coordinates $(x, y, z)$ and potentially additional attributes.
- The `crop_volume` is defined by its geometric bounds.

**Steps:**

1. **Cropping:**
   - Iterate through all points in the input point cloud.
   - For each point $p$, check if it lies within the `crop_volume`.
   - If $p$ is inside the volume, keep it; otherwise, discard it.
   - [G] The check depends on the volume type (e.g., for an AABB, check $x_{min} \le x \le x_{max}$, etc.).

2. **Downsampling:**
   - Determine the number of points remaining after cropping.
   - If `down_sample_method` is `'voxel'`:
     - Create a voxel grid with spacing `voxel_size`.
     - For each point, compute its voxel index: $v_x = \lfloor x / \text{voxel\_size} \rfloor$, $v_y = \lfloor y / \text{voxel\_size} \rfloor$, $v_z = \lfloor z / \text{voxel\_size} \rfloor$.
     - [G] Use a hash map or sparse grid to store the first point encountered for each unique voxel index.
     - The output is the set of representative points (one per occupied voxel).
   - If `down_sample_method` is `'uniform'`:
     - If the number of points exceeds `max_size`, randomly select `max_size` points from the cropped set.
     - Otherwise, keep all cropped points.
     - [G] Random selection should be uniform and without replacement.

3. **Transformation Application (Optional):**
   - If `init_trans` is provided (typically in registration workflows), apply the transformation to the source point cloud *before* cropping and downsampling [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def crop_and_downsample(point_cloud, crop_volume, down_sample_method='voxel', 
                        voxel_size=0.01, max_size=None, init_trans=None):
    """
    Crops a point cloud using a volume and downsamples it.
    
    Args:
        point_cloud: List of points with position and attributes.
        crop_volume: Object defining the spatial bounds.
        down_sample_method: 'voxel' or 'uniform'.
        voxel_size: Size of voxels for voxel downsampling.
        max_size: Max points for uniform downsampling.
        init_trans: 4x4 transformation matrix to apply to source.
        
    Returns:
        Downsampled point cloud.
    """
    # Step 1: Apply initial transformation if provided
    if init_trans is not None:
        transformed_points = []
        for p in point_cloud:
            # Apply 4x4 matrix to 3D point (homogeneous coordinates)
            p_hom = [p.x, p.y, p.z, 1.0]
            res = matmul(init_trans, p_hom)
            transformed_points.append(Point(res[0], res[1], res[2], p.attributes))
        point_cloud = transformed_points

    # Step 2: Crop
    cropped_points = []
    for p in point_cloud:
        if is_inside(p, crop_volume):
            cropped_points.append(p)

    # Step 3: Downsample
    if down_sample_method == 'voxel':
        return voxel_downsample(cropped_points, voxel_size)
    elif down_sample_method == 'uniform':
        if max_size is None:
            # Default value not specified in source, using a large number or no limit
            # Note: Source mentions max_size default is 4 * MAX_POINT_NUMBER but value is unknown
            return cropped_points
        if len(cropped_points) > max_size:
            return uniform_sample(cropped_points, max_size)
        else:
            return cropped_points
    else:
        raise ValueError("Unknown down_sample_method")

def voxel_downsample(points, voxel_size):
    """
    Keeps one point per voxel.
    """
    voxel_map = {} # Maps (vx, vy, vz) -> point
    for p in points:
        # Compute voxel coordinates
        vx = floor(p.x / voxel_size)
        vy = floor(p.y / voxel_size)
        vz = floor(p.z / voxel_size)
        key = (vx, vy, vz)
        if key not in voxel_map:
            voxel_map[key] = p
    return list(voxel_map.values())

def uniform_sample(points, max_size):
    """
    Randomly samples max_size points from points.
    """
    import random
    indices = random.sample(range(len(points)), max_size)
    return [points[i] for i in indices]

def is_inside(point, volume):
    """
    Check if point is inside the crop volume.
    Implementation depends on volume type (AABB, Sphere, etc.).
    """
    # Placeholder for specific volume check logic
    pass
```

**Parameters and Defaults:**
- `down_sample_method`: `'voxel'` (default) or `'uniform'` [S1, unofficial].
- `voxel_size`: `0.01` (default) [S1, unofficial].
- `max_size`: Default is `4 * MAX_POINT_NUMBER`, but `MAX_POINT_NUMBER` value is not specified in the source snippet [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Input:** If cropping removes all points, the output is empty.
- **Precision:** Floating-point precision in voxel index calculation can cause points near voxel boundaries to be assigned inconsistently if not handled carefully.
- **Large Point Clouds:** Uniform downsampling requires random sampling, which can be memory-intensive if the cloud is too large to fit in memory for shuffling. Voxel downsampling is generally more scalable for dense clouds.
- **Transformation Order:** In registration, the transformation must be applied *before* cropping/downsampling to ensure the source cloud is aligned with the target before density reduction [S1, unofficial].

## Subtopics

### Farthest Point Sampling (FPS)
A method that operates in continuous Euclidean space, defining convolution weights at specific kernel points influenced by a linear correlation function [S2]. It is used to extract hierarchical features by performing downsampling, often enhancing robustness to variation in point density when combined with grid subsampling or partition-based grid pooling [S2]. Unlike voxel downsampling, FPS selects points based on geometric distance, ensuring maximum spread, which is beneficial for feature extraction in deep learning models [S2].

### Voxel Downsampling Registration
Performs ICP registration between source and ground truth target point clouds using voxel downsampling, with an initial transformation [S1, unofficial]. The process involves: 1. Printing voxel size and threshold, setting verbosity to Debug if requested. 2. Downsampling the source point cloud using `crop_and_downsample` with method `'voxel'` and the given `voxel_size`, applying `init_trans` if present. 3. Downsampling the target point cloud similarly. 4. Performing ICP registration with a threshold, identity initial guess (or accumulated transformation), Point-to-Point estimation, and convergence criteria (tolerance $10^{-6}$, max iterations `max_itr`). 5. Updating the final transformation by multiplying the ICP result with `init_trans`. 6. Returning the registration result [S1, unofficial].

### Uniform Registration
Performs ICP registration between source and ground truth target point clouds using uniform downsampling, with an initial transformation [S1, unofficial]. The process involves: 1. Printing threshold, setting verbosity to Debug if requested. 2. Downsampling the source point cloud using `crop_and_downsample` with method `'uniform'` and `init_trans`. 3. Downsampling the target point cloud similarly. 4. Performing ICP registration with the same parameters as Voxel Downsampling Registration. 5. Updating the final transformation. 6. Returning the registration result [S1, unofficial]. The `max_size` parameter defaults to `4 * MAX_POINT_NUMBER`, but its exact value is not specified [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Cropping is $O(N)$ where $N$ is the number of points. Voxel downsampling is $O(N)$ with a hash map lookup. Uniform downsampling is $O(N)$ for sampling if using reservoir sampling or $O(N \log N)$ if sorting/shuffling. ICP registration is typically $O(M \log M)$ or $O(M)$ depending on the nearest neighbor search, where $M$ is the downsampled point count.
- **Memory Complexity:** $O(N)$ for storing the point cloud. Voxel downsampling requires $O(K)$ memory for the voxel grid, where $K$ is the number of occupied voxels.
- **Strengths:** Efficiently reduces point cloud size while preserving spatial distribution (voxel) or random representation (uniform). Integrates seamlessly with ICP registration pipelines [S1, unofficial].
- **Limitations:** Voxel downsampling may lose fine details if `voxel_size` is too large. Uniform downsampling may remove important features if they are sparse.
- **When to Use:** Use voxel downsampling when preserving spatial density is important for registration or voxelization. Use uniform downsampling when a random subset is sufficient or when computational simplicity is preferred.

## Background
The method is part of the preprocessing pipeline for 3D point cloud processing, particularly in the context of 2D Gaussian Splatting and ICP registration [S1, unofficial]. It is implemented in the `2d-gaussian-splatting` repository [S1, unofficial]. The use of point clouds as a fundamental primitive for 3D representation has a long history in computer graphics [S2].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
