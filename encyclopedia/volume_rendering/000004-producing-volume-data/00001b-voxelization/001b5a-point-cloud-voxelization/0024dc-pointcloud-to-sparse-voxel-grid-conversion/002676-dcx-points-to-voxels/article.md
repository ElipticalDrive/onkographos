# DCX Points-to-Voxels

Also called: Custom Voxel Grid Implementation, Hair Point Voxelization, Point Cloud Voxelization via Space Index Array, Voxelization of Point Clouds

## Overview

DCX Points-to-Voxels converts a set of 3D points into a sparse voxel grid representation by mapping each point to a discrete voxel index within a defined bounding box, storing occupancy in a 3D integer array, and computing the centroid of points within each occupied voxel [S1, S2, S3, unofficial]. It differs from simple averaging by prioritizing points with higher depth confidence in its aggregation step, which helps mitigate noise and outliers in regions with uncertain geometry [S1, unofficial]. It serves as the final aggregation step in the DCX pipeline, refining the raw point cloud into a structured sparse representation suitable for downstream rendering or reconstruction tasks [S1, unofficial], contrasting with direct occupancy mapping in [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md) or multi-plane approaches in [Multi-Plane Point Cloud Voxelization and Weight Aggregation](12101-multi-plane-point-cloud-voxelization-and-weight-aggregation.md).

## How it works

**Inputs:**
1. Point cloud data: A list of $N$ points, where each point $P_i$ has coordinates $(x_i, y_i, z_i)$ and optionally color $(r_i, g_i, b_i)$ [S1, S3, unofficial].
2. Resolution: The side length of a voxel in world units ($res$) [S2, S3, unofficial].
3. Bounding Box: Defined by minimum $(min_x, min_y, min_z)$ and maximum $(max_x, max_y, max_z)$ coordinates. This can be computed from the data (AABB) or fixed (CUSTOM) [S2, unofficial].

**Outputs:**
1. Sparse Voxel Grid: A data structure mapping voxel indices to aggregated properties (centroid position, color, point count) [S1, unofficial].
2. Occupancy Array: A 3D integer array indicating which voxels are occupied [S2, unofficial].

**Data Structures:**
* **Voxels Map:** A hash map where the key is a linear voxel ID (integer) and the value is a structure containing: `count` (integer, number of points), `sum_pos` (vector of size 3, sum of coordinates), and `sum_color` (vector of size 3, sum of RGB colors, if applicable) [S1, S3, unofficial].
* **Space Index Array:** A fixed-size 3D integer array `space_Index` initialized to 0. In the reference implementation, dimensions are hardcoded to `[192][128][128]` regardless of the bounding box mode [S2, unofficial].
* **Auxiliary Map:** In some implementations, a secondary map tracks voxel existence and counts during the merge phase to avoid redundant lookups [S1, unofficial].

**Parameters:**
* `resolution` ($res$): Voxel size. Typical values: 0.011, 0.016, or 0.1 [S2, S3, unofficial].
* `density_threshold`: Minimum number of points required to create a voxel. Default: 1 [S3, unofficial].
* `color_levels`: Number of quantization levels for color reduction. 0=off, 8/16/32=quantize. Default: 0 [S3, unofficial].

**Steps:**

1. **Compute Bounding Box and Grid Dimensions:**
   * If AABB mode: Compute $min\_bounds = \min(P)$ and $max\_bounds = \max(P)$ across all points [S2, unofficial].
   * If CUSTOM mode: Use predefined bounds. For example, X: $[-total\_reg\_X/2, total\_reg\_X/2]$, Y: $[0, total\_reg\_Y]$, Z: $[-total\_reg\_Z/2, total\_reg\_Z/2]$ [S2, unofficial].
   * Calculate grid dimensions:
     $$ D_x = \lceil \frac{max\_bounds_x - min\_bounds_x}{res} \rceil + 1 $$
     $$ D_y = \lceil \frac{max\_bounds_y - min\_bounds_y}{res} \rceil + 1 $$
     $$ D_z = \lceil \frac{max\_bounds_z - min\_bounds_z}{res} \rceil + 1 $$
   * In specific implementations, dimensions are hardcoded based on total range and resolution (e.g., 128x192x128) [S2, unofficial].

2. **Initialize Data Structures:**
   * Initialize `space_Index` array with zeros [S2, unofficial].
   * Initialize `Voxels Map` as empty [S1, unofficial].

3. **Map Points to Voxels and Aggregate:**
   For each point $P_i = (x_i, y_i, z_i)$:
   a. **Compute Voxel Coordinates:**
      $$ i = \lfloor \frac{x_i - min\_bounds_x}{res} \rfloor $$
      $$ j = \lfloor \frac{y_i - min\_bounds_y}{res} \rfloor $$
      $$ k = \lfloor \frac{z_i - min\_bounds_z}{res} \rfloor $$
      *Note: Some implementations use `ceil` for X and `floor` for Y/Z relative to bounding box minimums [S2, unofficial].*
   b. **Check Bounds:**
      If $i < 0$ or $i \ge D_x$ or $j < 0$ or $j \ge D_y$ or $k < 0$ or $k \ge D_z$, skip the point [G].
   c. **Compute Linear Index:**
      $$ idx = i + j \cdot D_x + k \cdot D_x \cdot D_y $$
      *Note: Specific implementations may use different stride rules, e.g., $idx = i \cdot 1 + j \cdot 128 + k \cdot (128 \cdot 192)$ if dimensions are fixed [S2, unofficial].*
   d. **Update Voxels Map:**
      If $idx$ not in `Voxels Map`:
      * Create entry: `count = 1`, `sum_pos = (x_i, y_i, z_i)`, `sum_color = (r_i, g_i, b_i)` (if color exists) [S1, S3, unofficial].
      * Insert into `Voxels Map`.
      Else:
      * Retrieve existing entry.
      * `count = count + 1`
      * `sum_pos = sum_pos + (x_i, y_i, z_i)` [S1, S3, unofficial]
      * `sum_color = sum_color + (r_i, g_i, b_i)` (if color exists) [S3, unofficial]
      * Update `Voxels Map[idx]`.
   e. **Mark Occupancy:**
      Set `space_Index[i][j][k] = 1`.
      *Warning: Some implementations swap X and Y indices and apply a -1 offset on X: `space_Index[j][i-1][k] = 1`. This can cause out-of-bounds access if $i=0$ [S2, unofficial]. Use standard $[i][j][k]$ unless replicating the specific bug/quirk.*

4. **Filter Voxels:**
   Iterate through `Voxels Map`. If `count < density_threshold`, remove the voxel from the map [S3, unofficial].

5. **Compute Centroids:**
   For each remaining voxel in `Voxels Map`:
   $$ center\_pos = \frac{sum\_pos}{count} $$
   $$ avg\_color = \frac{sum\_color}{count} $$
   *   If `color_levels` > 0, quantize `avg_color` to the specified number of levels [S3, unofficial].
   *   Store `center_pos` and `avg_color` in the final voxel structure [S1, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from collections import defaultdict

def dcx_points_to_voxels(points, resolution, bbox_min, bbox_max, density_threshold=1, color_levels=0):
    """
    Convert points to sparse voxels.
    
    Args:
        points: List of tuples (x, y, z, r, g, b). If no color, r,g,b can be None.
        resolution: Voxel size.
        bbox_min: Tuple (min_x, min_y, min_z).
        bbox_max: Tuple (max_x, max_y, max_z).
        density_threshold: Min points per voxel.
        color_levels: Quantization levels for color.
        
    Returns:
        voxels: Dict mapping (i, j, k) to {'pos': (cx, cy, cz), 'color': (r, g, b), 'count': int}
        space_index: 3D list/array of 0s and 1s.
    """
    min_x, min_y, min_z = bbox_min
    max_x, max_y, max_z = bbox_max
    
    # Calculate grid dimensions
    Dx = math.ceil((max_x - min_x) / resolution) + 1
    Dy = math.ceil((max_y - min_y) / resolution) + 1
    Dz = math.ceil((max_z - min_z) / resolution) + 1
    
    # Initialize space index array
    space_index = [[[0 for _ in range(Dz)] for _ in range(Dy)] for _ in range(Dx)]
    
    # Voxels map: key=(i,j,k), value={'sum_pos': [x,y,z], 'sum_color': [r,g,b], 'count': int}
    voxels_map = defaultdict(lambda: {'sum_pos': [0.0, 0.0, 0.0], 'sum_color': [0, 0, 0], 'count': 0})
    
    has_color = points[0][3] is not None if points else False
    
    for p in points:
        x, y, z = p[0], p[1], p[2]
        
        # Compute voxel coordinates
        i = math.floor((x - min_x) / resolution)
        j = math.floor((y - min_y) / resolution)
        k = math.floor((z - min_z) / resolution)
        
        # Check bounds
        if not (0 <= i < Dx and 0 <= j < Dy and 0 <= k < Dz):
            continue
            
        # Update map
        voxel_data = voxels_map[(i, j, k)]
        voxel_data['sum_pos'][0] += x
        voxel_data['sum_pos'][1] += y
        voxel_data['sum_pos'][2] += z
        voxel_data['count'] += 1
        
        if has_color:
            voxel_data['sum_color'][0] += p[3]
            voxel_data['sum_color'][1] += p[4]
            voxel_data['sum_color'][2] += p[5]
            
        # Mark occupancy
        space_index[i][j][k] = 1
        
    # Filter and compute centroids
    final_voxels = {}
    for (i, j, k), data in voxels_map.items():
        if data['count'] < density_threshold:
            continue
            
        cx = data['sum_pos'][0] / data['count']
        cy = data['sum_pos'][1] / data['count']
        cz = data['sum_pos'][2] / data['count']
        
        r = data['sum_color'][0] / data['count']
        g = data['sum_color'][1] / data['count']
        b = data['sum_color'][2] / data['count']
        
        # Quantize color if needed
        if color_levels > 0:
            step = 256 / color_levels
            r = int(round(r / step) * step)
            g = int(round(g / step) * step)
            b = int(round(b / step) * step)
            # Clamp
            r = max(0, min(255, r))
            g = max(0, min(255, g))
            b = max(0, min(255, b))
            
        final_voxels[(i, j, k)] = {
            'pos': (cx, cy, cz),
            'color': (r, g, b),
            'count': data['count']
        }
        
    return final_voxels, space_index
```

**Edge Cases and Pitfalls:**
* **Index Out of Bounds:** In specific implementations, the index mapping `space_Index[ijk1][ijk0-1][ijk2]` involves a -1 offset on the X-index. If `ijk0` is 0, this results in an out-of-bounds access (index -1) [S2, unofficial]. Always check bounds before accessing the occupancy array [G].
* **Bounding Box Mismatch:** If points fall outside the defined bounding box (especially in CUSTOM mode), the computed indices may be negative or exceed array bounds, leading to undefined behavior [S2, unofficial]. Ensure the bounding box encompasses all input points [G].
* **Precision:** Using `floor` vs `ceil` can shift voxel assignment by 1. Be consistent with the source implementation [S2, unofficial].
* **Empty Voxels:** Voxels with fewer points than `density_threshold` are discarded. If `density_threshold` is too high, the grid may become too sparse [S3, unofficial].

## Subtopics

### [Weighted Voxel Center Computation from Points](11233-weighted-voxel-center-computation-from-points.md)
Computes the spatial center of each voxel by calculating the weighted mean of 3D points falling within that voxel, using depth confidence as weights. It differs from simple averaging by prioritizing points with higher depth confidence, which helps mitigate noise and outliers. Choose this when depth confidence data is available and noise reduction is critical.

## Operating characteristics

* **Time Complexity:** $O(N)$ for mapping points to voxels, where $N$ is the number of points. Aggregation is also $O(N)$. Filtering and centroid computation is $O(V)$, where $V$ is the number of occupied voxels. Overall: $O(N + V)$ [G].
* **Memory Complexity:** $O(V)$ for storing voxel data. The occupancy array is $O(D_x \cdot D_y \cdot D_z)$, which can be large if the bounding box is large and resolution is high, but it is sparse in practice if only occupied voxels are stored in the map [G].
* **Performance:** Efficient for large point clouds due to linear scan and hash map usage. GPU acceleration is possible by parallelizing point-to-voxel mapping [G].
* **Strengths:** Simple to implement, handles arbitrary point distributions, supports color aggregation.
* **Limitations:** Fixed resolution grid may waste memory if the scene is sparse or has varying density. Does not handle overlapping points within a voxel better than simple averaging (unless weighted variant is used) [S1, unofficial].
* **When to Use:** When a structured voxel representation is needed for rendering, reconstruction, or spatial queries. When point cloud data is dense enough to justify voxelization.
* **When Not to Use:** When the scene is extremely sparse and a sparse voxel octree or KD-tree would be more efficient. When high-frequency details are lost due to voxel averaging.

## Background

The method originates from unofficial implementations for converting point clouds (including 3D Gaussian Splatting and PLY formats) into sparse voxel grids for rendering in USD or other formats [S1, S3, unofficial]. It is often used in pipelines involving 3DGS to Voxel conversion, leveraging USD's PointInstancer for efficient rendering [S3, unofficial].

## Sources

[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S2] ericmaic/Point-Cloud-Voxelization, https://github.com/ericmaic/Point-Cloud-Voxelization, fetched 2026-10-27 (unofficial)
[S3] omolism/3DGS-to-Voxel-OpenUSD, https://github.com/omolism/3DGS-to-Voxel-OpenUSD, fetched 2026-10-27 (unofficial)
