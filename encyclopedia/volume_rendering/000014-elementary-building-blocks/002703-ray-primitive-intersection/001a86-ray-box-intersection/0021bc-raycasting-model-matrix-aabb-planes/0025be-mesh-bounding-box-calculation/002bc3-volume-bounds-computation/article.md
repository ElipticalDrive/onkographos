# Volume Bounds Computation

Also called: computeVolumeBounds

## Overview
Computes an Axis-Aligned Bounding Box (AABB) that encloses a set of 3D points, with optional padding [S1, unofficial]. Unlike [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md), which operates on vertex coordinates of explicit geometry, this method operates on arbitrary point clouds or sample sets [S1, unofficial].

## How it works
**Inputs:**
- `points`: An array of 3D points, where each point is a tuple `[x, y, z]` of floating-point numbers [S1, unofficial].
- `padding`: A scalar value representing the padding to add to all sides of the bounding box in world units [S1, unofficial].

**Outputs:**
- `VolumeBounds`: An object containing:
  - `min`: `[number, number, number]` — The minimum coordinates of the AABB [S1, unofficial].
  - `max`: `[number, number, number]` — The maximum coordinates of the AABB [S1, unofficial].
  - `size`: `[number, number, number]` — The dimensions of the AABB (`max - min`) [S1, unofficial].
  - `center`: `[number, number, number]` — The center of the AABB (`(min + max) / 2`) [S1, unofficial].

**Data Structures:**
- `VolumeBounds`:
  - `min`: Array of 3 floats [S1, unofficial].
  - `max`: Array of 3 floats [S1, unofficial].
  - `size`: Array of 3 floats [S1, unofficial].
  - `center`: Array of 3 floats [S1, unofficial].

**Steps:**
1. Initialize `min` and `max` arrays to infinity and negative infinity, respectively, for all three axes (x, y, z) [G].
2. Iterate over all input points to find min and max x, y, z coordinates [S1, unofficial].
3. Apply configurable padding in world units to all sides [S1, unofficial].
4. Output `VolumeBounds` object containing `min`, `max`, `size`, and `center` [S1, unofficial].

**Formulas:**
- $size[i] = max[i] - min[i]$ for each axis $i \in \{0, 1, 2\}$ [G].
- $center[i] = (min[i] + max[i]) / 2.0$ for each axis $i \in \{0, 1, 2\}$ [G].

**Parameters:**
- `padding`: Configurable padding in world units. Default value not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- Empty point set: If `points` is empty, `min` remains infinity and `max` remains negative infinity. The implementation should handle this case, potentially by returning a default bounds or raising an error. Not given in the sources [S1, unofficial].
- Precision: Use floating-point arithmetic consistent with the input data type. Not given in the sources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volume_bounds(points, padding):
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

## Subtopics

### [Silhouette-based Bounding Box Initialization](5448-silhouette-based-bounding-box-initialization.md)
Computes an Axis-Aligned Bounding Box (AABB) that encloses the visible geometry of a scene by projecting ground truth masks from training cameras. It serves as a robust initialization step for volume rendering, particularly when sparse representations are used, ensuring the bounding volume tightly fits the object of interest rather than the entire scene space. It differs from simple min-max point accumulation by using binary silhouette masks to determine visibility, making it resilient to background noise and sparse sampling artifacts.

### Silhouette-Based Bounding Box Estimation
Computes a 3D bounding box for the scene by backprojecting silhouette masks from multiple training views to find the tightest volume containing all visible geometry [S2, unofficial]. It differs from the basic point-based computation by using camera projections to infer 3D extent from 2D masks.

**Steps:**
1. Iterate over training views [S2, unofficial].
2. For each view, extract binary silhouette mask [S2, unofficial].
3. Backproject mask pixels into 3D space using camera intrinsics/extrinsics [S2, unofficial].
4. Accumulate 3D points/volumes [S2, unofficial].
5. Compute min/max bounds along x, y, z axes [S2, unofficial].
6. Apply `silhouetteRadius` expansion [S2, unofficial].
7. Store bounds in `scene.bbox_depth/width/height_min/max` [S2, unofficial].

**Parameters:**
- `train_cameras`: Array of camera poses [S2, unofficial].
- `groundTruth`: Array of ground truth images with masks [S2, unofficial].
- `scene`: Object scene structure to store computed bounds [S2, unofficial].
- `datasetSize/viewCount`: Number of training views [S2, unofficial].
- `silhouetteRadius`: Radius parameter for silhouette expansion [S2, unofficial].
- `skipIndex/skips`: Indices and strides for subsampling views [S2, unofficial].

### Octree-based Bounding Volume Computation
An alternative strategy to compute bounding volumes by decomposing space into an octree and computing the convex hull from straddling cubes [S3]. It differs from grid-based approaches by adaptively refining space only where the signed distance function indicates proximity to the surface.

**Steps:**
1. Start from a large cube [S3].
2. Recursively subdivide the cube if the value $f(c)$ at its center $c$ is greater than the length of the diagonal of the cube [S3].
3. Compute the convex hull $H$ from the eight vertices of the detected straddling cubes [S3].

**Pitfalls:**
- Coplanar points in the final point set can slow convex hull computation, as all points may contribute to the convex envelope [S3].

**Comparison:**
- Requires significantly more signed distance function (SDF) evaluations than Sphere Carving for good convex hulls, but may converge faster for shapes with significant anisotropy in $f$ values [S3].

## Operating characteristics
- **Time Complexity:** O(N) for the basic method, where N is the number of points, due to a single pass over the input array [S1, unofficial]. Complexity for silhouette-based estimation depends on the number of views and pixels per mask, but exact bounds are not given in the sources [S2, unofficial].
- **Memory Complexity:** O(1) for the basic method, as it only stores min/max values and derived fields [S1, unofficial]. Silhouette-based estimation requires storing intermediate 3D points or volumes, but exact bounds are not given in the sources [S2, unofficial].
- **Strengths:** Simple, predictable performance for static datasets [S1, unofficial].
- **Limitations:** Basic method is sensitive to outliers in the point cloud, as it computes exact min/max without filtering [G]. Silhouette-based method requires accurate camera poses and masks [G].
- **When to use:** Use the basic method for known point clouds [G]. Use silhouette-based estimation when visibility from training views is the primary constraint [G].

## Background
Originated in the context of 3D IDW interpolation volume and voxel visualization [S1, unofficial]. The octree-based variant traces to adaptive space decomposition strategies for implicit surfaces [S3]. Not given in the sources for other historical details [S1, S2, unofficial][S3].

## Sources
[S1] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
