# Hard Voxelization with Capacity Limits

Also called: points2voxel, Point Binning

## Overview
Hard Voxelization with Capacity Limits converts a set of 3D points into a fixed-size voxel grid representation by grouping points into discrete voxels, storing their coordinates, and enforcing strict limits on the number of points per voxel and the total number of voxels [S1, unofficial][S2, unofficial]. It differs from its sibling [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md) by explicitly managing capacity constraints via rejection counters and chunk-based memory management, often used in scenarios requiring bounded memory usage or streaming data processing [S2, unofficial].

## How it works
**Inputs:**
- `points`: Array of 3D coordinates, shape $(N, 3)$ or $(N, 4)$ if labels are included.
- `voxel_size`: List of 3 floats $[s_x, s_y, s_z]$, defining the physical size of a single voxel in each dimension [S1, unofficial].
- `coors_range`: List of 6 floats $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$, defining the bounding box of the voxel grid [S1, unofficial].
- `max_points`: Integer, maximum number of points stored per voxel. Default is 35 [S1, unofficial] or 5 [S1, unofficial].
- `max_voxels`: Integer, maximum number of voxels this function creates. Default is 20000 [S1, unofficial].
- `reverse_index`: Boolean, whether to return coordinates in reversed order $(z, y, x)$ instead of $(x, y, z)$. Default is True [S1, unofficial].
- `specific_category`: Optional list of integers, filters points by label. Points with label 31 are explicitly skipped [S1, unofficial].

**Outputs:**
- `voxels`: Tensor of shape $(M, \text{max\_points}, 3)$ or $(M, \text{max\_points}, 4)$, containing the coordinates of points in active voxels. $M$ is the number of active voxels [S1, unofficial].
- `coors`: Tensor of shape $(M, 3)$, containing the voxel coordinates for each active voxel [S1, unofficial].
- `num_points_per_voxel`: Tensor of shape $(M)$, containing the count of points in each active voxel [S1, unofficial].

**Data Structures:**
- `coor_to_voxelidx`: 3D array of shape $(D, H, W)$ or $(W, H, D)$ depending on `reverse_index`, initialized to -1. Maps voxel coordinates to a linear voxel index in the output arrays [S1, unofficial].
- `voxels`: Pre-allocated array of shape $(\text{max\_voxels}, \text{max\_points}, 3)$, initialized to zeros [S1, unofficial].
- `num_points_per_voxel`: Array of shape $(\text{max\_voxels})$, initialized to zeros [S1, unofficial].

**Steps:**
1. **Compute Grid Shape:** Calculate the dimensions of the voxel grid:
   $$
   \text{voxelmap\_shape} = \text{tuple}\left(\left\lfloor \frac{\text{coors\_range}[3:] - \text{coors\_range}[:3]}{\text{voxel\_size}} \right\rfloor\right)
   $$
   If `reverse_index` is True, reverse the shape to $(W, H, D)$ [S1, unofficial].

2. **Initialize Structures:**
   - Initialize `coor_to_voxelidx` with -1 [S1, unofficial].
   - Initialize `num_points_per_voxel` with 0 [S1, unofficial].
   - Initialize `voxels` with 0 [S1, unofficial].
   - Set `voxel_num = 0` [S1, unofficial].

3. **Process Points:** Iterate through each point $p_i$ in `points`:
   a. **Validation:** If any coordinate of $p_i$ is not finite (NaN or Inf), skip the point [S2, unofficial].
   b. **Coordinate Conversion:** Compute the voxel coordinate $c = (c_x, c_y, c_z)$:
      $$
      c_j = \left\lfloor \frac{p_{i,j} - \text{coors\_range}[j]}{\text{voxel\_size}[j]} \right\rfloor \quad \text{for } j \in \{0, 1, 2\}
      $$
      [S1, unofficial]. Alternatively, implemented as $c = \lfloor p \cdot (1 / \text{resolution}) \rfloor$ [S2, unofficial].
   c. **Boundary Check:** If $c_j < 0$ or $c_j \ge \text{voxelmap\_shape}[j]$ for any $j$, skip the point [S1, unofficial].
   d. **Label Filtering:** If `specific_category` is provided and the point's label is not in the list (or is 31), skip the point [S1, unofficial].
   e. **Index Mapping:**
      - If `reverse_index` is True, reverse the coordinate order: $c' = (c_z, c_y, c_x)$ [S1, unofficial].
      - Look up `voxelidx = coor_to_voxelidx[c']`.
      - If `voxelidx == -1`:
        - If `voxel_num >= max_voxels`, skip the point [S1, unofficial].
        - Else, assign `voxelidx = voxel_num`, increment `voxel_num`, set `coor_to_voxelidx[c'] = voxelidx`, and initialize `coors[voxelidx] = c'` [S1, unofficial].
   f. **Point Insertion:**
      - If `num_points_per_voxel[voxelidx] < max_points`:
        - Store $p_i$ into `voxels[voxelidx, num_points_per_voxel[voxelidx]]`.
        - Increment `num_points_per_voxel[voxelidx]` [S1, unofficial].
      - Else:
        - Increment `rejected_count` [S2, unofficial].

4. **Post-Processing (Optional Label Aggregation):**
   - If labels are present, sort points by label index [S1, unofficial].
   - Assign point labels to `voxel[x, y, z, label_count[x, y, z]]` [S1, unofficial].
   - Increment `label_count[x, y, z]` [S1, unofficial].
   - After processing all points, replace multi-label voxels with the most frequent label using `np.argmax(np.bincount(labels[labels!=0]))` [S1, unofficial].
   - Reset `label_count` entries equal to `max_points` to 0 [S1, unofficial].
   - Convert `voxel` to `np.float64` [S1, unofficial].

5. **Return:** Slice `voxels`, `coors`, and `num_points_per_voxel` to `voxel_num` and return [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_voxelization_with_capacity_limits(points, voxel_size, coors_range, max_points=35, max_voxels=20000, reverse_index=True, specific_category=None):
    # 1. Compute grid shape
    voxelmap_shape = tuple(np.round((np.array(coors_range[3:]) - np.array(coors_range[:3])) / np.array(voxel_size)).astype(np.int32))
    if reverse_index:
        voxelmap_shape = voxelmap_shape[::-1]
    
    # 2. Initialize structures
    coor_to_voxelidx = np.full(voxelmap_shape, -1, dtype=np.int32)
    num_points_per_voxel = np.zeros(max_voxels, dtype=np.int32)
    voxels = np.zeros((max_voxels, max_points, points.shape[1]), dtype=points.dtype)
    coors = np.zeros((max_voxels, 3), dtype=np.int32)
    voxel_num = 0
    
    # 3. Process points
    for i in range(points.shape[0]):
        p = points[i]
        
        # Validation
        if not np.all(np.isfinite(p[:3])):
            continue
            
        # Coordinate conversion
        voxel_coord = np.floor((p[:3] - np.array(coors_range[:3])) / np.array(voxel_size)).astype(np.int32)
        
        # Boundary check
        if np.any(voxel_coord < 0) or np.any(voxel_coord >= voxelmap_shape):
            continue
            
        # Label filtering
        if specific_category is not None:
            if p.shape[0] > 3:
                label = int(p[3])
                if label == 31 or label not in specific_category:
                    continue
        
        # Index mapping
        if reverse_index:
            coor = voxel_coord[::-1]
        else:
            coor = voxel_coord
            
        voxelidx = coor_to_voxelidx[coor[0], coor[1], coor[2]]
        
        if voxelidx == -1:
            if voxel_num >= max_voxels:
                continue
            voxelidx = voxel_num
            voxel_num += 1
            coor_to_voxelidx[coor[0], coor[1], coor[2]] = voxelidx
            coors[voxelidx] = coor
            
        # Point insertion
        if num_points_per_voxel[voxelidx] < max_points:
            voxels[voxelidx, num_points_per_voxel[voxelidx]] = p
            num_points_per_voxel[voxelidx] += 1
        else:
            # Rejected point
            pass
            
    # 4. Slice and return
    return voxels[:voxel_num], coors[:voxel_num], num_points_per_voxel[:voxel_num]
```

**Edge Cases and Pitfalls:**
- Points outside `coors_range` are skipped [S1, unofficial].
- If `voxel_num` reaches `max_voxels`, subsequent points are dropped [S1, unofficial]. To ensure randomness in which points are dropped, points should be shuffled before calling [S1, unofficial].
- If a voxel reaches `max_points`, additional points are dropped [S1, unofficial].
- In streaming implementations, chunk readiness must be polled with a timeout (e.g., 10ms) to avoid infinite blocking during disk I/O [S2, unofficial].

## Subtopics

### [Probabilistic Raytracing Occupancy Update](12215-probabilistic-raytracing-occupancy-update.md)
Updates voxel occupancy probabilities using raytracing hits and misses, adding confidence measures to deterministic hard voxelization. Choose when smoother integration of sparse data is needed.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of points, assuming constant-time hash map or array lookup for voxel coordinates.
- **Memory Complexity:** $O(\text{max\_voxels} \times \text{max\_points})$ for the output arrays, plus $O(\text{voxelmap\_shape})$ for the coordinate-to-index map [S1, unofficial].
- **Performance:** Performance is limited by memory bandwidth when writing to the `voxels` array and by cache efficiency of the `coor_to_voxelidx` map [S1, unofficial].
- **Strengths:** Bounded memory usage, deterministic output, simple implementation.
- **Limitations:** Points are dropped if capacity limits are reached, which can lead to information loss in dense regions [S1, unofficial].
- **When to use:** When memory is constrained, or when processing streaming data with known bounds [S2, unofficial].
- **When not to use:** When preserving all point information is critical, or when the point distribution is highly variable and exceeds capacity limits frequently.

## Background
Hard Voxelization with Capacity Limits is a standard technique in 3D point cloud processing, particularly in autonomous driving and robotics for creating voxel-based representations for neural networks [S1, unofficial]. It is commonly used in frameworks like MMDetection3D and custom voxelization libraries like voxlvoxl [S1, unofficial][S2, unofficial].

## Sources
[S1] ZZY816/VoxelSplat, https://github.com/ZZY816/VoxelSplat, fetched 2026-10-27 (unofficial)
[S2] goguma426/voxlvoxl, https://github.com/goguma426/voxlvoxl, fetched 2026-10-28 (unofficial)
