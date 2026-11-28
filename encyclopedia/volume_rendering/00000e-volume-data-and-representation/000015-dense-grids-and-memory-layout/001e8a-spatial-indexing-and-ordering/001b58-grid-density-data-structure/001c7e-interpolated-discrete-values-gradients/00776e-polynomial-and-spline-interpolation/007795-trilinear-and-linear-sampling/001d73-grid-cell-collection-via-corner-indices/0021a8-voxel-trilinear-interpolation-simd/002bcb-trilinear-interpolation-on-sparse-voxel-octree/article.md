# Trilinear Interpolation on Sparse Voxel Octree


## Overview
Trilinear interpolation on a Sparse Voxel Octree (SVO) samples scalar or vector fields stored in the leaf nodes of an octree structure by mapping 3D query points to grid coordinates, identifying the eight surrounding voxel neighbors, and computing a weighted sum of their payload values [S1, unofficial]. It differs from dense grid trilinear interpolation by replacing direct array indexing with an octree traversal to locate leaf payloads, allowing efficient storage of sparse data while maintaining continuous interpolation [S1, unofficial]. It builds on [Voxel Trilinear Interpolation (SIMD)](2564-voxel-trilinear-interpolation-simd.md) but replaces the dense memory layout with hierarchical node descriptors and leaf indices.

## How it works
**Inputs:**
- `octree`: Structure containing `NodeDescriptor` array, `leaf_payload_indices` array, `max_depth`, and `root_bounds` (min/max `glm::vec3`).
- `points`: Vector of 3D points (`glm::vec3`) to sample.
- `payload`: Pointer to raw payload data array (contiguous storage for leaf data).
- `payload_rows`: Total number of rows (leaf entries) in the payload array.
- `channels`: Number of scalar values per payload row (e.g., 1 for grayscale, 3 for RGB).
- `fill_value`: Value returned for points outside root bounds or in empty voxels.

**Outputs:**
- `outputs`: Vector of size `points.size() * channels`, initialized to `fill_value`.

**Data Structures:**
- `NodeDescriptor`: Contains `child_mask` (bitmask of existing children), `leaf_mask` (bitmask of leaf children), `child_base` (offset to internal children), and `payload_base` (offset to leaf payloads) [S1, unofficial].
- `leaf_payload_indices`: Array mapping leaf IDs to indices in the `payload` array [S1, unofficial].

**Steps:**
1. **Validate Inputs:** Ensure `max_depth` is in $[0, 30]$, `channels > 0`, `payload` is not null if `payload_rows > 0`, and all leaf payload indices are within `payload_rows` [S1, unofficial]. If branching mode is Wide4, throw `ValidationError` [S1, unofficial].
2. **Initialize Output:** Set `outputs` to size `points.size() * channels` and fill with `fill_value` [S1, unofficial].
3. **Calculate Grid Size:** Compute `grid_size = 2^{max_depth}` [S1, unofficial].
4. **Loop Over Points:** For each point `p` in `points`:
   a. **Bounds Check:** If `p` is not inside `root_bounds` (i.e., $p.x < min\_x$ or $p.x \ge max\_x$, etc.), skip to next point [S1, unofficial].
   b. **Normalize and Grid Convert:**
      $$ normalized = \frac{p - root\_bounds.min}{root\_bounds.max - root\_bounds.min} $$
      $$ grid = normalized \times grid\_size - 0.5 $$
   c. **Base Coordinates:**
      $$ base = \lfloor grid \rfloor $$
      $$ frac = grid - base $$
   d. **Interpolation Loop:** Initialize `accumulated = 0.0`. Loop `dz` from 0 to 1, `dy` from 0 to 1, `dx` from 0 to 1:
      i. **Weights:**
         $$ wz = \begin{cases} 1.0 - frac.z & \text{if } dz=0 \\ frac.z & \text{if } dz=1 \end{cases} $$
         $$ wy = \begin{cases} 1.0 - frac.y & \text{if } dy=0 \\ frac.y & \text{if } dy=1 \end{cases} $$
         $$ wx = \begin{cases} 1.0 - frac.x & \text{if } dx=0 \\ frac.x & \text{if } dx=1 \end{cases} $$
      ii. **Total Weight:**
          $$ weight = wx \times wy \times wz $$
          If `weight == 0`, continue to next iteration [S1, unofficial].
      iii. **Neighbor Coordinate:**
           $$ neighbor = base + (dx, dy, dz) $$
      iv. **Lookup Payload:**
          - If `neighbor` is outside grid bounds $[0, grid\_size)$, set `payload_index = -1`.
          - Else, call `lookup_payload_index_by_voxel(octree, neighbor, payload_rows)` to get `payload_index` [S1, unofficial].
          - If `payload_index >= 0`, set `neighbor_payload = payload + payload_index * channels`.
          - Else, set `neighbor_payload = nullptr` [S1, unofficial].
      v. **Accumulate:** Loop `c` from 0 to `channels - 1`:
          - If `neighbor_payload` is `nullptr`, `value = fill_value`.
          - Else, `value = neighbor_payload[c]`.
          - `accumulated[c] += weight * (value - fill_value)` [S1, unofficial].
   e. **Store Result:** For `c` from 0 to `channels - 1`:
      $$ outputs[point\_index \times channels + c] = fill\_value + accumulated[c] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_trilinear(octree, points, payload, payload_rows, channels, fill_value):
    # Validate inputs
    if not (0 <= octree.max_depth <= 30):
        raise ValueError("max_depth must be in [0, 30]")
    if channels <= 0:
        raise ValueError("channels must be > 0")
    if payload_rows > 0 and payload is None:
        raise ValueError("payload cannot be null if payload_rows > 0")
    
    # Initialize output
    num_points = len(points)
    outputs = [fill_value] * (num_points * channels)
    
    grid_size = 2 ** octree.max_depth
    root_min = octree.root_bounds.min
    root_max = octree.root_bounds.max
    extent = root_max - root_min
    
    for point_idx, p in enumerate(points):
        # Bounds check
        if not (root_min.x <= p.x < root_max.x and
                root_min.y <= p.y < root_max.y and
                root_min.z <= p.z < root_max.z):
            continue
            
        # Normalize and convert to grid coordinates
        normalized_x = (p.x - root_min.x) / extent.x
        normalized_y = (p.y - root_min.y) / extent.y
        normalized_z = (p.z - root_min.z) / extent.z
        
        grid_x = normalized_x * grid_size - 0.5
        grid_y = normalized_y * grid_size - 0.5
        grid_z = normalized_z * grid_size - 0.5
        
        # Base integer coordinates and fractional offsets
        base_x = int(math.floor(grid_x))
        base_y = int(math.floor(grid_y))
        base_z = int(math.floor(grid_z))
        
        frac_x = grid_x - base_x
        frac_y = grid_y - base_y
        frac_z = grid_z - base_z
        
        accumulated = [0.0] * channels
        
        for dz in range(2):
            wz = 1.0 - frac_z if dz == 0 else frac_z
            for dy in range(2):
                wy = 1.0 - frac_y if dy == 0 else frac_y
                for dx in range(2):
                    wx = 1.0 - frac_x if dx == 0 else frac_x
                    
                    weight = wx * wy * wz
                    if weight == 0:
                        continue
                    
                    neighbor_x = base_x + dx
                    neighbor_y = base_y + dy
                    neighbor_z = base_z + dz
                    
                    # Check grid bounds
                    if not (0 <= neighbor_x < grid_size and
                            0 <= neighbor_y < grid_size and
                            0 <= neighbor_z < grid_size):
                        payload_index = -1
                    else:
                        # Lookup payload index via octree traversal
                        payload_index = lookup_payload_index_by_voxel(
                            octree, (neighbor_x, neighbor_y, neighbor_z), payload_rows
                        )
                    
                    if payload_index >= 0:
                        neighbor_payload = payload[payload_index * channels : (payload_index + 1) * channels]
                    else:
                        neighbor_payload = None
                    
                    for c in range(channels):
                        if neighbor_payload is None:
                            value = fill_value
                        else:
                            value = neighbor_payload[c]
                        
                        accumulated[c] += weight * (value - fill_value)
        
        # Store result
        for c in range(channels):
            outputs[point_idx * channels + c] = fill_value + accumulated[c]
            
    return outputs

def lookup_payload_index_by_voxel(octree, voxel, payload_rows):
    """
    Traverses the SVO to find the payload index for a given voxel coordinate.
    Returns -1 if the voxel is empty or out of bounds.
    """
    nodes = octree.nodes
    leaf_indices = octree.leaf_payload_indices
    max_depth = octree.max_depth
    root_min = octree.root_bounds.min
    root_max = octree.root_bounds.max
    extent = root_max - root_min
    
    if len(nodes) == 0:
        return -1
        
    # Calculate voxel indices relative to root
    scale = 2 ** max_depth
    voxel_x = int(math.floor(((voxel[0] - root_min[0]) / extent[0]) * scale))
    voxel_y = int(math.floor(((voxel[1] - root_min[1]) / extent[1]) * scale))
    voxel_z = int(math.floor(((voxel[2] - root_min[2]) / extent[2]) * scale))
    
    # Check if voxel is within the root bounds grid
    if not (0 <= voxel_x < scale and 0 <= voxel_y < scale and 0 <= voxel_z < scale):
        return -1
        
    node_index = 0
    depth_remaining = max_depth
    
    while True:
        if node_index >= len(nodes):
            return -1
            
        descriptor = nodes[node_index]
        
        # Determine child index based on depth
        child_depth = depth_remaining - 1
        x_bits = (voxel_x >> child_depth) & 3
        y_bits = (voxel_y >> child_depth) & 3
        z_bits = (voxel_z >> child_depth) & 3
        child_index = x_bits | (y_bits << 2) | (z_bits << 4)
        
        child_bit = 1 << child_index
        
        # Check if child exists
        if not (descriptor.child_mask() & child_bit):
            return -1
            
        # Check if child is a leaf
        if descriptor.leaf_mask() & child_bit:
            # Calculate leaf ID
            leaf_id = descriptor.payload_base() + prefix_rank(descriptor.leaf_mask(), child_index)
            if leaf_id >= payload_rows:
                return -1
            return leaf_indices[leaf_id]
            
        # Move to internal child
        internal_mask = descriptor.internal_child_mask()
        node_index = descriptor.child_base() + prefix_rank(internal_mask, child_index)
        depth_remaining -= 1
        
        if depth_remaining <= 0:
            return -1

def prefix_rank(mask, index):
    """Count number of set bits in mask before the given index."""
    count = 0
    for i in range(index):
        if mask & (1 << i):
            count += 1
    return count
```

**Parameters:**
- `octree`: Sparse Voxel Octree structure with node descriptors and leaf payload indices [S1, unofficial].
- `points`: Vector of 3D points (`glm::vec3`) [S1, unofficial].
- `payload`: Pointer to raw payload data array [S1, unofficial].
- `payload_rows`: Total number of rows in the payload array [S1, unofficial].
- `channels`: Number of scalar values per payload row [S1, unofficial].
- `fill_value`: Value for points outside bounds or empty voxels [S1, unofficial].

**Edge Cases and Pitfalls:**
- Points outside the root bounds are skipped, leaving their output as `fill_value` [S1, unofficial].
- Wide4 branching mode is not supported and throws a `ValidationError` [S1, unofficial].
- Precision issues may arise if `grid_size` is very large; however, the sources do not specify a maximum safe `max_depth` beyond the validation range $[0, 30]$ [S1, unofficial].

## Subtopics

### Trilinear Interpolation with Adaptive Subdivision Handling
Interpolation method for irradiance volumes using seven LERPs of the spherical harmonic coefficients [S2, unofficial]. It searches the tree for the voxel containing the object’s centroid, interpolates surrounding samples (trilinear), and finds a weighted sum of surrounding samples weighted by $1/distance$ [S2, unofficial]. To prevent popping artifacts when transitioning between finely and coarsely subdivided voxels, it continues using samples from subdivided neighbors for interpolation, requiring each octree node to store pointers to samples on each face [S2, unofficial]. Naïve trilinear interpolation can produce popping artifacts in adaptively subdivided volumes because some sample data is suddenly ignored as an object moves from fine to coarse voxels [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(\log N)$ per point for octree traversal, where $N$ is the number of nodes, plus constant time for the 8-neighbor interpolation [S1, unofficial].
**Memory Complexity:** $O(1)$ auxiliary space per point, excluding the storage of the octree and payload [S1, unofficial].
**Measured Performance:** Not given in the sources.
**Scaling:** Scales with the number of points and the depth of the octree. Sparse data allows for efficient sampling in large volumes with few occupied voxels [S1, unofficial].
**Strengths:** Efficiently handles sparse 3D data without allocating memory for empty space [S1, unofficial].
**Limitations:** Slower than dense grid lookup due to octree traversal overhead; not suitable for uniformly dense data where dense arrays are more cache-friendly [S1, unofficial].
**When to Use:** When the volume data is sparse or has varying resolution (adaptive subdivision) [S1, S2, unofficial].
**When Not to Use:** When the volume is dense and uniformly sampled, or when extreme performance is required and memory is not a constraint [S1, unofficial].

## Background
The method originates from implementations of Sparse Voxel Octrees for volume rendering and ray tracing, as seen in the `sparse-voxels` repository [S1, unofficial]. The concept of trilinear interpolation on octrees for irradiance volumes was discussed in the context of game graphics at GDC 2005, highlighting issues with adaptive subdivision [S2, unofficial].

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S2] PDF Irradiance Volumes for Games, https://chrisoat.com/papers/Oat_GDC2005_IrradianceVolumesForGames.pdf, fetched 2026-10-28 (unofficial)
