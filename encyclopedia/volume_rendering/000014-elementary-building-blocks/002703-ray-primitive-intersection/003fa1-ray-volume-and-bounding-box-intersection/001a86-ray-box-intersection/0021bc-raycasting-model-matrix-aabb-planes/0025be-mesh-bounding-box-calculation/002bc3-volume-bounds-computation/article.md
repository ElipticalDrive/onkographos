# Volume Bounds Computation

Also called: computeVolumeBounds

## Overview
Volume Bounds Computation computes an Axis-Aligned Bounding Box (AABB) that encloses a set of 3D points, optionally applying padding to ensure the volume fully contains the geometry with a safety margin [S1, unofficial]. It serves as the foundational step for initializing voxel grids, spatial indices, and rendering volumes in implicit and explicit volume rendering pipelines. Unlike [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md), which operates on mesh vertices, this method generalizes to any point cloud or sampled data structure. It differs from [Silhouette-based Bounding Box Initialization](5448-silhouette-based-bounding-box-initialization.md) by requiring explicit 3D coordinates rather than inferring bounds from 2D projections.

## How it works
### Inputs and Outputs
**Inputs:**
- `points`: An array of 3D points, where each point is a tuple `[x, y, z]`.
- `padding`: A scalar value representing the padding to add to all sides of the bounding box in world units [S1, unofficial].

**Outputs:**
- `VolumeBounds`: An object containing:
  - `min`: `[number, number, number]` — The minimum coordinates of the AABB.
  - `max`: `[number, number, number]` — The maximum coordinates of the AABB.
  - `size`: `[number, number, number]` — The dimensions of the AABB (`max - min`).
  - `center`: `[number, number, number]` — The center of the AABB (`(min + max) / 2`).

### Data Structures
**VolumeBounds Object Layout:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```typescript
type VolumeBounds = {
  min: [number, number, number],
  max: [number, number, number],
  size: [number, number, number],
  center: [number, number, number]
}
```
[S1, unofficial]

### Algorithm Steps
1. **Initialize Bounds:** Set `min` to `[Infinity, Infinity, Infinity]` and `max` to `[-Infinity, -Infinity, -Infinity]`.
2. **Iterate Points:** For each point `p` in `points`:
   - Update `min.x = min(min.x, p.x)`, `min.y = min(min.y, p.y)`, `min.z = min(min.z, p.z)`.
   - Update `max.x = max(max.x, p.x)`, `max.y = max(max.y, p.y)`, `max.z = max(max.z, p.z)`.
3. **Apply Padding:**
   - `min.x -= padding`, `min.y -= padding`, `min.z -= padding`.
   - `max.x += padding`, `max.y += padding`, `max.z += padding`.
4. **Compute Derived Fields:**
   - `size.x = max.x - min.x`, `size.y = max.y - min.y`, `size.z = max.z - min.z`.
   - `center.x = (min.x + max.x) / 2`, `center.y = (min.y + max.y) / 2`, `center.z = (min.z + max.z) / 2`.
5. **Return:** Construct and return the `VolumeBounds` object.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volume_bounds(points: list, padding: float) -> dict:
    # Initialize min and max with extreme values
    min_val = [float('inf'), float('inf'), float('inf')]
    max_val = [float('-inf'), float('-inf'), float('-inf')]
    
    # Iterate over all input points to find min and max coordinates
    for point in points:
        min_val[0] = min(min_val[0], point[0])
        min_val[1] = min(min_val[1], point[1])
        min_val[2] = min(min_val[2], point[2])
        
        max_val[0] = max(max_val[0], point[0])
        max_val[1] = max(max_val[1], point[1])
        max_val[2] = max(max_val[2], point[2])
        
    # Apply configurable padding in world units to all sides
    min_val[0] -= padding
    min_val[1] -= padding
    min_val[2] -= padding
    
    max_val[0] += padding
    max_val[1] += padding
    max_val[2] += padding
    
    # Compute size and center
    size = [
        max_val[0] - min_val[0],
        max_val[1] - min_val[1],
        max_val[2] - min_val[2]
    ]
    
    center = [
        (min_val[0] + max_val[0]) / 2.0,
        (min_val[1] + max_val[1]) / 2.0,
        (min_val[2] + max_val[2]) / 2.0
    ]
    
    # Output VolumeBounds object
    return {
        "min": min_val,
        "max": max_val,
        "size": size,
        "center": center
    }
```

### Parameters
- `padding`: Configurable padding in world units. Default value not specified in sources. [S1, unofficial]

### Edge Cases and Pitfalls
- **Empty Input:** If `points` is empty, `min` remains `[Infinity, ...]` and `max` remains `[-Infinity, ...]`. The resulting size will be negative, and center will be `NaN`. Implementers should check for empty input and handle it (e.g., return a default zero-sized box or throw an error) [G].
- **Precision:** Using `Infinity` for initialization is standard, but for very large coordinate values, floating-point precision issues may arise. [G]
- **Degenerate Input:** If all points are identical, the size will be `[2*padding, 2*padding, 2*padding]` after padding is applied.

## Subtopics

### [Silhouette-based Bounding Box Initialization](5448-silhouette-based-bounding-box-initialization.md)
Silhouette-based Bounding Box Initialization computes an Axis-Aligned Bounding Box (AABB) and initializes the adaptive learning rate using silhouette information from training cameras and ground truth masks. It serves as the entry point for volume bounds computation when explicit scene geometry is unavailable.

### Bounding Cube Creation
Not given in the sources.

### Character Bounding Box Initialization
Not given in the sources.

### Host-side Bounding Box Computation
Not given in the sources.

### Silhouette-Based Bounding Box Estimation
Computes a 3D bounding box for the scene by backprojecting silhouette masks from multiple training views to find the tightest volume containing all visible geometry [S2, unofficial].
**Steps:**
1. Iterate over training views.
2. For each view, extract binary silhouette mask.
3. Backproject mask pixels into 3D space using camera intrinsics/extrinsics.
4. Accumulate 3D points/volumes.
5. Compute min/max bounds along x, y, z axes.
6. Apply `silhouetteRadius` expansion.
7. Store bounds in `scene.bbox_depth/width/height_min/max`.
**Parameters:**
- `train_cameras`: Array of camera poses.
- `groundTruth`: Array of ground truth images with masks.
- `scene`: Object scene structure to store computed bounds.
- `datasetSize/viewCount`: Number of training views.
- `silhouetteRadius`: Radius parameter for silhouette expansion.
- `skipIndex/skips`: Indices and strides for subsampling views.
[S2, unofficial]

### Octree-based Bounding Volume Computation
An alternative strategy to compute bounding volumes by decomposing space into an octree and computing the convex hull from straddling cubes [S3].
**Steps:**
1. Start from a large cube.
2. Recursively subdivide the cube if the value $f(c)$ at its center $c$ is greater than the length of the diagonal of the cube.
3. Compute the convex hull $H$ from the eight vertices of the detected straddling cubes.
**Pitfalls:**
- Coplanar points in the final point set can slow convex hull computation, as all points may contribute to the convex envelope.
**Comparison:**
- Requires significantly more SDF evaluations than Sphere Carving for good convex hulls, but may converge faster for shapes with significant anisotropy in $f$ values.
[S3]

### Robust Bounding Box Computation
A method to compute a bounding box for a 3DGS scene by clipping outliers using percentiles, preventing distant 'floater' splats from dominating the AABB [S4, unofficial].
**Steps:**
1. Collect all X, Y, and Z coordinates of the splat centers into separate arrays.
2. Sort each coordinate array in ascending order.
3. Calculate the lower index as $\lfloor p \times (N - 1) \rfloor$ and the upper index as $\lceil (1 - p) \times (N - 1) \rceil$, where $N$ is the number of splats.
4. Construct the bounding box using the coordinates at the lower and upper indices for each axis.
**Parameters:**
- `bboxPercentile`: The fraction of splats to clip from each end of the distribution per axis. Typical value: 0.02 (clip outer 2%).
**Complexity:**
- $O(N \log N)$ due to sorting, where $N$ is the number of splats.
[S4, unofficial]

## Operating characteristics
- **Time Complexity:** $O(N)$ for the basic method, where $N$ is the number of input points [S1, unofficial]. The robust percentile-based method is $O(N \log N)$ due to sorting [S4, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space for the basic method, excluding the input and output storage. The robust method requires $O(N)$ space to store coordinate arrays.
- **Measured Performance:** Not given in the sources.
- **Scaling:** The basic method scales linearly with the number of points. The robust method scales with $N \log N$.
- **Strengths:** Simple, fast, and predictable for static datasets [S1, unofficial]. The robust method handles outliers effectively.
- **Limitations:** The basic method is sensitive to outliers. The silhouette-based method requires camera poses and masks. The octree method requires many SDF evaluations.
- **When to Use:** Use the basic method for known point clouds. Use the robust method for 3DGS scenes with potential outliers. Use the silhouette-based method when only 2D training data is available. Use the octree method for SDF-based bounding volumes.
- **When Not to Use:** Do not use the basic method if outliers are present and not handled. Do not use the octree method if SDF evaluations are expensive and the shape is simple.

## Background
The concept of computing bounding boxes for volume rendering is fundamental to efficient ray traversal and voxel grid construction. The basic `computeVolumeBounds` function is part of the IDW (Inverse Distance Weighted) interpolation volume pipeline, designed for predictable performance on static datasets [S1, unofficial]. The robust percentile-based method was developed for 3D Gaussian Splatting (3DGS) carving to handle "floater" splats [S4, unofficial]. The silhouette-based method is used in NeRF-like training pipelines to initialize scene bounds from 2D data [S2, unofficial]. The octree-based approach is related to Sphere Carving techniques for SDF bounding volumes [S3].

## Sources
[S1] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
[S4] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
