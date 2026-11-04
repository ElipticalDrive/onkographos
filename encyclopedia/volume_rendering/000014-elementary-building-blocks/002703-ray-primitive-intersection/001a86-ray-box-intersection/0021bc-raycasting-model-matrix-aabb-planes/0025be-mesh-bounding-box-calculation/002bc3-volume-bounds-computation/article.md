# Volume Bounds Computation

Also called: computeVolumeBounds

## Overview
Computes an Axis-Aligned Bounding Box (AABB) that encloses a set of 3D points, with optional padding [S1, unofficial]. It serves as the foundational spatial extent definition for volume rendering grids, ensuring the voxelization domain tightly fits the data of interest. Unlike [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md), which operates on vertex coordinates of explicit geometry, this method operates on arbitrary point clouds or sample sets, making it suitable for implicit representations or scattered data [S1, unofficial].

## How it works
**Inputs:**
- `points`: An array of 3D points, where each point is a tuple `[x, y, z]` of floating-point numbers.
- `padding`: A scalar value representing the padding to add to all sides of the bounding box in world units. Default is 0.0 [S1, unofficial].

**Outputs:**
- `VolumeBounds`: An object containing:
  - `min`: `[number, number, number]` — The minimum coordinates of the AABB.
  - `max`: `[number, number, number]` — The maximum coordinates of the AABB.
  - `size`: `[number, number, number]` — The dimensions of the AABB (`max - min`).
  - `center`: `[number, number, number]` — The center of the AABB (`(min + max) / 2`) [S1, unofficial].

**Data Structures:**
- `VolumeBounds`:
  - `min`: Array of 3 floats.
  - `max`: Array of 3 floats.
  - `size`: Array of 3 floats.
  - `center`: Array of 3 floats [S1, unofficial].

**Steps:**
1. Initialize `min` and `max` arrays to infinity and negative infinity, respectively, for all three axes (x, y, z).
2. Iterate over each point `p` in `points`:
   a. For each axis `i` in {0, 1, 2}:
      i. Update `min[i] = min(min[i], p[i])`.
      ii. Update `max[i] = max(max[i], p[i])`.
3. Apply padding:
   a. For each axis `i` in {0, 1, 2}:
      i. `min[i] = min[i] - padding`.
      ii. `max[i] = max[i] + padding`.
4. Compute `size[i] = max[i] - min[i]` for each axis `i`.
5. Compute `center[i] = (min[i] + max[i]) / 2.0` for each axis `i`.
6. Return the `VolumeBounds` object [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volume_bounds(points, padding=0.0):
    # Initialize min and max to extreme values
    min_coords = [float('inf'), float('inf'), float('inf')]
    max_coords = [float('-inf'), float('-inf'), float('-inf')]
    
    # Iterate over all input points to find min and max x, y, z coordinates
    for point in points:
        for i in range(3):
            if point[i] < min_coords[i]:
                min_coords[i] = point[i]
            if point[i] > max_coords[i]:
                max_coords[i] = point[i]
                
    # Apply configurable padding in world units to all sides
    for i in range(3):
        min_coords[i] = min_coords[i] - padding
        max_coords[i] = max_coords[i] + padding
        
    # Compute size and center
    size_coords = [max_coords[i] - min_coords[i] for i in range(3)]
    center_coords = [(min_coords[i] + max_coords[i]) / 2.0 for i in range(3)]
    
    # Output VolumeBounds object
    return {
        'min': min_coords,
        'max': max_coords,
        'size': size_coords,
        'center': center_coords
    }
```

**Parameters:**
- `padding`: Configurable padding in world units. Default value not specified, but 0.0 is implied if not provided [S1, unofficial].

**Edge Cases and Pitfalls:**
- Empty point set: If `points` is empty, `min` remains infinity and `max` remains negative infinity. The implementation should handle this case, potentially by returning a default bounds or raising an error. Not given in the sources [S1, unofficial].
- Precision: Use floating-point arithmetic consistent with the input data type. Not given in the sources [S1, unofficial].

## Subtopics

### [Silhouette-based Bounding Box Initialization](5448-silhouette-based-bounding-box-initialization.md)
Computes an Axis-Aligned Bounding Box (AABB) that encloses the visible geometry of a scene by projecting ground truth masks from training cameras. It serves as a robust initialization step for volume rendering, particularly when sparse representations are used, ensuring the bounding volume tightly fits the object of interest rather than the entire scene space. It differs from simple min-max point accumulation by using binary silhouette masks to determine visibility, making it resilient to background noise and sparse sampling artifacts.

### Silhouette-Based Bounding Box Estimation
Computes a 3D bounding box for the scene by backprojecting silhouette masks from multiple training views to find the tightest volume containing all visible geometry. It differs from the basic point-based computation by using camera projections to infer 3D extent from 2D masks. [S2, unofficial]

**Steps:**
1. Iterate over training views.
2. For each view, extract binary silhouette mask.
3. Backproject mask pixels into 3D space using camera intrinsics/extrinsics.
4. Accumulate 3D points/volumes.
5. Compute min/max bounds along x, y, z axes.
6. Apply `silhouetteRadius` expansion.
7. Store bounds in `scene.bbox_depth/width/height_min/max` [S2, unofficial].

**Parameters:**
- `train_cameras`: Array of camera poses [S2, unofficial].
- `groundTruth`: Array of ground truth images with masks [S2, unofficial].
- `scene`: Object scene structure to store computed bounds [S2, unofficial].
- `datasetSize/viewCount`: Number of training views [S2, unofficial].
- `silhouetteRadius`: Radius parameter for silhouette expansion [S2, unofficial].
- `skipIndex/skips`: Indices and strides for subsampling views [S2, unofficial].

## Operating characteristics
- **Time Complexity:** O(N) for the basic method, where N is the number of points [S1, unofficial]. For silhouette-based estimation, complexity depends on the number of views and pixels per mask [S2, unofficial].
- **Memory Complexity:** O(1) for the basic method, as it only stores min/max values [S1, unofficial]. Silhouette-based estimation requires storing intermediate 3D points or volumes [S2, unofficial].
- **Strengths:** Simple, predictable performance for static datasets [S1, unofficial]. Silhouette-based method is robust to background noise and sparse sampling [S2, unofficial].
- **Limitations:** Basic method is sensitive to outliers in the point cloud [S1, unofficial]. Silhouette-based method requires accurate camera poses and masks [S2, unofficial].
- **When to use:** Use the basic method for known point clouds. Use silhouette-based estimation for sparse or noisy data where visibility is key [S1, S2, unofficial].

## Background
Originated in the context of 3D IDW interpolation volume and voxel visualization [S1, unofficial]. Also used in NeRF-based methods like Clear-plenoxels for scene initialization [S2, unofficial].

## Sources
[S1] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
