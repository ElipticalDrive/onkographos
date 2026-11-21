# Volume Bounds Computation

Also called: computeVolumeBounds

## Overview
Volume Bounds Computation computes an Axis-Aligned Bounding Box (AABB) that encloses a set of 3D points, optionally applying padding to ensure the volume strictly contains the geometry with a safety margin [S1, unofficial]. It serves as the foundational step for initializing voxel grids, splat carvers, and raymarchers by defining the spatial extent of the scene [S1, S3, unofficial]. Unlike [Bounding Cube Creation](4007-bounding-cube-creation.md), which forces a cubic aspect ratio, or [Character Bounding Box Initialization](4016-character-bounding-box-initialization.md), which targets specific skeletal hierarchies, this method produces a tight rectangular prism tailored to the input point distribution [S1, unofficial]. It builds on [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md) by generalizing the min/max vertex logic to arbitrary point clouds rather than just mesh vertices [S1, unofficial].

## How it works
**Inputs:**
- `points`: An array of 3D points, where each point is a tuple or vector `[x, y, z]`.
- `padding`: A scalar value representing the padding to add to all sides of the bounding box in world units [S1, unofficial]. If not specified, defaults to 0.

**Outputs:**
- `VolumeBounds`: An object containing:
  - `min`: `[number, number, number]` — The minimum coordinates of the AABB.
  - `max`: `[number, number, number]` — The maximum coordinates of the AABB.
  - `size`: `[number, number, number]` — The dimensions of the AABB (`max - min`).
  - `center`: `[number, number, number]` — The geometric center of the AABB.

**Data Structures:**
The `VolumeBounds` object is a flat structure with four fields, each being a 3-element array of floating-point numbers [S1, unofficial].

**Algorithm Steps:**
1. Initialize `min` to `[Infinity, Infinity, Infinity]` and `max` to `[-Infinity, -Infinity, -Infinity]` [G].
2. Iterate through each point `p` in `points`:
   a. Update `min.x = min(min.x, p.x)`, `min.y = min(min.y, p.y)`, `min.z = min(min.z, p.z)`.
   b. Update `max.x = max(max.x, p.x)`, `max.y = max(max.y, p.y)`, `max.z = max(max.z, p.z)`.
3. Apply padding:
   a. `min.x -= padding`, `min.y -= padding`, `min.z -= padding`.
   b. `max.x += padding`, `max.y += padding`, `max.z += padding`.
4. Compute `size`:
   a. `size.x = max.x - min.x`
   b. `size.y = max.y - min.y`
   c. `size.z = max.z - min.z`
5. Compute `center`:
   a. `center.x = (min.x + max.x) / 2`
   b. `center.y = (min.y + max.y) / 2`
   c. `center.z = (min.z + max.z) / 2`
6. Return `VolumeBounds` object with `min`, `max`, `size`, and `center`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volume_bounds(points: list[tuple[float, float, float]], padding: float = 0.0) -> dict:
    # Initialize min and max with extreme values
    min_val = [float('inf'), float('inf'), float('inf')]
    max_val = [float('-inf'), float('-inf'), float('-inf')]

    # Iterate over all input points to find min and max coordinates
    for p in points:
        if p[0] < min_val[0]: min_val[0] = p[0]
        if p[1] < min_val[1]: min_val[1] = p[1]
        if p[2] < min_val[2]: min_val[2] = p[2]
        
        if p[0] > max_val[0]: max_val[0] = p[0]
        if p[1] > max_val[1]: max_val[1] = p[1]
        if p[2] > max_val[2]: max_val[2] = p[2]

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

    return {
        "min": min_val,
        "max": max_val,
        "size": size,
        "center": center
    }
```

**Parameters:**
- `padding`: Configurable padding in world units. Typical value: 0.0 for tight bounds, >0.0 for safety margins [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Input:** If `points` is empty, `min` remains `[inf, inf, inf]` and `max` remains `[-inf, -inf, -inf]`. The caller must check for this case before using the result [G].
- **Precision:** Using `float('inf')` ensures correct initialization regardless of the scale of the input data, avoiding underflow/overflow issues with arbitrary initial values [G].
- **Degenerate Input:** If all points are identical, `size` will be `[0, 0, 0]`. Padding ensures the box has non-zero volume if `padding > 0` [S1, unofficial].

## Subtopics

### [Silhouette-based Bounding Box Initialization](5448-silhouette-based-bounding-box-initialization.md)
Silhouette-based Bounding Box Initialization computes an Axis-Aligned Bounding Box (AABB) and initializes the adaptive learning rate using silhouette information from training cameras and ground truth masks. It serves as the entry point for volume bounds computation when explicit scene geometry is unavailable. It differs from the standard point-cloud method by deriving bounds from 2D image masks rather than 3D points.

### [Octree-based Bounding Volume Computation](6270-octree-based-bounding-volume-computation.md)
Octree-based Bounding Volume Computation computes a convex bounding volume for a signed distance field (SDF) by adaptively decomposing space into an octree and extracting the convex hull from cubes that straddle the surface. It serves as an alternative to Sphere Carving, trading higher SDF evaluation costs for potentially faster convergence on shapes with significant anisotropy in the field values. Unlike Sphere Carving, which provides strict bounding guarantees, this approach does not guarantee that the resulting volume encloses the entire surface.

### Robust Bounding Box Computation
Robust Bounding Box Computation is a method to compute a bounding box for a 3DGS scene by clipping outliers using percentiles, preventing distant 'floater' splats from dominating the AABB [S3, unofficial]. It differs from the standard method by ignoring extreme values in the coordinate distribution [S3, unofficial].
- **Definition:** Computes a bounding box by sorting splat centers and selecting coordinates at percentile boundaries [S3, unofficial].
- **Steps:**
  1. Collect all X, Y, and Z coordinates of the splat centers into separate arrays [S3, unofficial].
  2. Sort each coordinate array in ascending order [S3, unofficial].
  3. Calculate the lower index as `floor(p * (N - 1))` and the upper index as `ceil((1 - p) * (N - 1))`, where N is the number of splats and p is `bboxPercentile` [S3, unofficial].
  4. Construct the bounding box using the coordinates at the lower and upper indices for each axis [S3, unofficial].
- **Parameters:** `bboxPercentile`: The fraction of splats to clip from each end of the distribution per axis. Typical value: 0.02 (clip outer 2%) [S3, unofficial].
- **Complexity:** O(N log N) due to sorting, where N is the number of splats [S3, unofficial].

### Oriented Bounding Box (OBB) Computation
Oriented Bounding Box (OBB) Computation computes oriented bounding boxes for 3D instances by analyzing the spatial distribution of assigned points [S4]. It differs from AABB methods by allowing rotation to fit the data more tightly [S4].
- **Definition:** Derives oriented bounding boxes based on the PCA of the convex hull of points belonging to each instance [S4].
- **Steps:**
  1. Compute point-wise instance assignments in 3D space [S4].
  2. For each instance, compute the convex hull of all points belonging to it [S4].
  3. Apply Principal Component Analysis (PCA) to the convex hull points to determine orientation and extent [S4].
  4. Derive oriented bounding boxes from the PCA results [S4].

## Operating characteristics
- **Time Complexity:** O(N) for the standard AABB computation, where N is the number of input points [S1, unofficial]. O(N log N) for the Robust Bounding Box Computation due to sorting [S3, unofficial].
- **Memory Complexity:** O(N) to store the input points and coordinate arrays for the robust method [S3, unofficial]. The standard method uses O(1) auxiliary space [S1, unofficial].
- **Strengths:** Simple, fast, and predictable performance for static datasets [S1, unofficial]. The robust variant effectively handles outliers common in neural rendering scenes (e.g., 3DGS floaters) [S3, unofficial].
- **Limitations:** The standard AABB may be too loose if the scene has significant anisotropy or outliers [S3, unofficial]. Silhouette-based methods require accurate camera poses and masks [S2, unofficial]. OBB computation is more expensive and complex [S4].
- **When to use:** Use standard AABB for general point clouds and mesh bounds [S1, unofficial]. Use robust bounding box for 3D Gaussian Splatting scenes with potential outliers [S3, unofficial]. Use silhouette-based when no 3D geometry is available [S1, unofficial]. Use OBB when tight fitting for oriented instances is required [S4].

## Background
Volume Bounds Computation is a fundamental technique in computer graphics for defining the spatial extent of a scene or object. It is widely used in volume rendering to initialize voxel grids, splat carvers, and raymarchers [S1, S3, unofficial]. The method is often a precursor to more complex bounding volume hierarchies or spatial partitioning structures [S1, unofficial]. In the context of neural rendering and 3D Gaussian Splatting, robust variants have been developed to handle the specific challenges of outlier splats [S3, unofficial]. Silhouette-based methods are common in NeRF-like pipelines where explicit geometry is not available [S2, unofficial].

## Sources
[S1] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
[S4] SUDS: Scalable Urban Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.01191, fetched 2026-10-26
