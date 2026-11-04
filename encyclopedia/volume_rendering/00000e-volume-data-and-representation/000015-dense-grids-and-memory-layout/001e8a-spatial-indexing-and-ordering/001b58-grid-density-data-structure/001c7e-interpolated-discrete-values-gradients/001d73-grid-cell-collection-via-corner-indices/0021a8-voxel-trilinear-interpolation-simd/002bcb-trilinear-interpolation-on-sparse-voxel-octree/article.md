# Trilinear Interpolation on Sparse Voxel Octree


## Overview
This method samples scalar or vector fields stored in the leaf nodes of a Sparse Voxel Octree (SVO) using trilinear interpolation [S1, unofficial]. It maps 3D query points to grid coordinates, identifies the 8 surrounding voxel neighbors, retrieves their payload values via the octree structure, and computes the weighted sum [S1, unofficial]. Unlike dense grid sampling, it navigates the hierarchical tree to access only occupied voxels, making it suitable for sparse volumetric data [S1, unofficial].

## How it works
**Inputs:**
- `octree`: Structure containing `NodeDescriptor` array, `leaf_payload_indices` array, `num_nodes`, `num_leaves`, `max_depth`, and `root_bounds` (min/max bounds).
- `points`: Vector of 3D points (`glm::vec3`) to sample.
- `payload`: Pointer to raw payload data array (contiguous storage for leaf data).
- `payload_rows`: Total number of rows (leaf entries) in the payload array.
- `channels`: Number of scalar values per payload row (e.g., 1 for grayscale, 3 for RGB).
- `fill_value`: Value returned for points outside root bounds or in empty voxels.

**Outputs:**
- `outputs`: Vector of size `points.size() * channels`, initialized to `fill_value`.

**Data Structures:**
- `NodeDescriptor`: Contains `child_mask` (bitmask of children), `leaf_mask` (bitmask of leaf children), `child_base` (offset to internal children), and `payload_base` (offset to leaf payloads). [S1, unofficial]
- `RootBounds`: `min_bound` and `max_bound` defining the spatial extent of the octree root. [S1, unofficial]

**Steps:**
1. **Validate Inputs:** Ensure `max_depth` is in $[0, 30]$, `channels > 0`, `payload` is not null if `payload_rows > 0`, and all `leaf_payload_indices` are within `payload_rows`. If branching mode is Wide4, throw `ValidationError` [S1, unofficial].
2. **Initialize Output:** Set `outputs` to size `points.size() * channels` filled with `fill_value`.
3. **Calculate Grid Size:** Compute `grid_size = 2^{max_depth}`.
4. **Loop Over Points:** For each point `p` in `points`:
   a. **Bounds Check:** If `p` is not inside `root_bounds` (i.e., $p.x < min\_bound.x$ or $p.x \ge max\_bound.x$, etc.), skip to next point.
   b. **Normalize Coordinates:** Compute normalized position:
      $$ normalized = \frac{p - min\_bound}{max\_bound - min\_bound} $$
   c. **Grid Coordinates:** Compute continuous grid coordinates:
      $$ grid = normalized \times grid\_size - 0.5 $$
   d. **Base Voxel:** Compute integer base coordinates:
      $$ base = \lfloor grid \rfloor $$
   e. **Fractional Offsets:** Compute fractional parts:
      $$ frac = grid - base $$
   f. **Trilinear Interpolation:** Initialize `output_index = point_index * channels`. Loop `dz` from 0 to 1, `dy` from 0 to 1, `dx` from 0 to 1:
      i. **Weights:**
         $$ wz = \begin{cases} 1.0 - frac.z & \text{if } dz=0 \\ frac.z & \text{if } dz=1 \end{cases} $$
         $$ wy = \begin{cases} 1.0 - frac.y & \text{if } dy=0 \\ frac.y & \text{if } dy=1 \end{cases} $$
         $$ wx = \begin{cases} 1.0 - frac.x & \text{if } dx=0 \\ frac.x & \text{if } dx=1 \end{cases} $$
      ii. **Total Weight:**
          $$ weight = wx \times wy \times wz $$
          If `weight == 0`, continue to next iteration.
      iii. **Neighbor Coordinate:**
           $$ neighbor = base + (dx, dy, dz) $$
      iv. **Lookup Payload Index:**
          - If `neighbor` is outside grid bounds $[0, grid\_size)$, set `payload_index = -1`.
          - Else, call `lookup_payload_index_by_voxel(octree, neighbor, payload_rows)` to get `payload_index`.
      v. **Retrieve Value:**
          - If `payload_index >= 0`, set `neighbor_payload = payload + payload_index * channels`.
          - Else, set `neighbor_payload = nullptr`.
          - For each `channel` in $[0, channels)$:
            - If `neighbor_payload` is `nullptr`, `value = fill_value`.
            - Else, `value = neighbor_payload[channel]`.
      vi. **Accumulate:**
          $$ outputs[output\_index + channel] += weight \times (value - fill\_value) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_trilinear(octree, points, payload, payload_rows, channels, fill_value):
    # Validate inputs
    if octree.max_depth < 0 or octree.max_depth > 30:
        raise ValueError("max_depth must be in [0, 30]")
    if channels <= 0:
        raise ValueError("channels must be > 0")
    if payload_rows > 0 and payload is None:
        raise ValueError("payload cannot be null if payload_rows > 0")
    
    # Initialize output
    num_points = len(points)
    outputs = [fill_value] * (num_points * channels)
    
    grid_size = 1 << octree.max_depth
    min_bound = octree.root_bounds.min
    max_bound = octree.root_bounds.max
    extent = max_bound - min_bound
    
    for point_index, point in enumerate(points):
        # Bounds check
        if not (min_bound.x <= point.x < max_bound.x and
                min_bound.y <= point.y < max_bound.y and
                min_bound.z <= point.z < max_bound.z):
            continue
            
        # Normalize and grid coordinates
        normalized = (point - min_bound) / extent
        grid = normalized * grid_size - 0.5
        
        # Base voxel and fractional offsets
        base = floor(grid)
        frac = grid - base
        
        output_index = point_index * channels
        
        for dz in range(2):
            wz = (1.0 - frac.z) if dz == 0 else frac.z
            for dy in range(2):
                wy = (1.0 - frac.y) if dy == 0 else frac.y
                for dx in range(2):
                    wx = (1.0 - frac.x) if dx == 0 else frac.x
                    
                    weight = wx * wy * wz
                    if weight == 0:
                        continue
                    
                    neighbor = base + (dx, dy, dz)
                    
                    # Check grid bounds
                    if not (0 <= neighbor.x < grid_size and
                            0 <= neighbor.y < grid_size and
                            0 <= neighbor.z < grid_size):
                        payload_index = -1
                    else:
                        payload_index = lookup_payload_index_by_voxel(octree, neighbor, payload_rows)
                    
                    # Retrieve value
                    if payload_index >= 0:
                        neighbor_payload = payload + payload_index * channels
                        for channel in range(channels):
                            value = neighbor_payload[channel]
                            outputs[output_index + channel] += weight * (value - fill_value)
                    else:
                        for channel in range(channels):
                            outputs[output_index + channel] += weight * (fill_value - fill_value) # Adds 0

    return outputs
```

**Parameters:**
- `octree`: Sparse voxel octree structure.
- `points`: Vector of 3D points.
- `payload`: Raw payload data array.
- `payload_rows`: Number of leaf entries.
- `channels`: Values per leaf.
- `fill_value`: Default value for empty/out-of-bounds.

**Edge Cases and Pitfalls:**
- Points outside root bounds are skipped, leaving output as `fill_value` [S1, unofficial].
- Wide4 branching mode is unsupported and throws `ValidationError` [S1, unofficial].
- Precision issues may arise if `grid_size` is very large; use appropriate floating-point types [G].

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$ where $N$ is the number of points and $D$ is the max depth of the octree for traversal, plus constant time for interpolation [S1, unofficial].
- **Memory Complexity:** $O(N \cdot C)$ for output storage, where $C$ is channels [S1, unofficial].
- **Strengths:** Efficient for sparse data; avoids sampling empty space [S1, unofficial].
- **Limitations:** Slower than dense grid sampling for dense data due to tree traversal overhead [S1, unofficial].
- **When to Use:** Sparse volumetric data, large scenes with empty regions [S1, unofficial].
- **When Not to Use:** Dense data, hardware with limited tree traversal support [S1, unofficial].

## Background
Originated in the `hectorpiteau/sparse-voxels` repository [S1, unofficial]. Used in GPU-based volume rendering for efficient sampling of sparse voxel octrees [S1, unofficial].

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
