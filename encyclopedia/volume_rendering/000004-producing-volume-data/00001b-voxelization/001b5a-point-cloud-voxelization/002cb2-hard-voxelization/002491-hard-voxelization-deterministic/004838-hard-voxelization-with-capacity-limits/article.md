# Hard Voxelization with Capacity Limits

Also called: Point Binning, Voxel Grid Update, Voxel Point Insertion & Binning

## Overview
This method inserts 3D points into a sparse voxel grid structure while enforcing a strict maximum capacity of points per voxel [S1, unofficial]. It differs from its parent [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md) by explicitly managing chunk readiness for disk I/O and tracking insertion/rejection statistics, making it suitable for streaming or sliding-window scenarios where data availability is asynchronous [S1, unofficial].

## How it works
**Inputs:**
- `points`: A list of 3D points, where each point $p$ has coordinates $(p.x, p.y, p.z)$ [S1, unofficial].
- `resolution`: The world-to-voxel scale factor (world units per voxel) [S1, unofficial].
- `max_points_per_voxel`: The integer limit on points stored in a single voxel [S1, unofficial].
- `sliding_window_size`: The size of the active chunk window [S1, unofficial].

**Outputs:**
- A voxel grid (sparse hash map) where each occupied voxel contains a list of points and a count.
- Statistics: `insert_count`, `rejected_count`, `out_of_range_count` [S1, unofficial].

**Data Structures:**
- `Chunk`: A shared pointer to a chunk of the voxel grid.
  - `accessor`: Pointer to the voxel map accessor [S1, unofficial].
  - `ready`: Atomic boolean, `true` if the chunk is loaded and not undergoing disk I/O [S1, unofficial].
  - `state`: Enum, `MODIFIED` if points were added [S1, unofficial].
  - `countdown`: Counter for chunk dismissal [S1, unofficial].
- `CellT`: The value type stored in a voxel.
  - `points`: Array of 3D points, size `max_points_per_voxel` [S1, unofficial].
  - `num_points`: Integer count of valid points in `points` [S1, unofficial].

**Steps:**
1. **Validate Point:** For each point $p$, check if $p.x$, $p.y$, and $p.z$ are finite. If any are not finite, skip the point [S1, unofficial].
2. **Compute Voxel Coordinate:** Calculate the voxel coordinate $v$ using the formula:
   $$ v = \lfloor p \cdot \frac{1}{\text{resolution}} \rfloor $$
   Implemented as `(p * inv_resolution_).array().floor().cast<int32_t>()` [S1, unofficial].
3. **Determine Chunk Index:** Calculate the chunk index corresponding to $v$ using `getCorrespondingChunkIdx(v)` [S1, unofficial].
4. **Check Window:** If the calculated chunk index is not a neighbor of the `current_source_chunk_idx_`, increment `out_of_range_count` and skip the point [S1, unofficial].
5. **Wait for Chunk:** Retrieve the `ChunkPtr` for the chunk index. Poll `chunk->ready`. If `ready` is false, sleep for 1ms. If the wait exceeds 10ms, log an error and return `false` [S1, unofficial].
6. **Access Voxel:** Call `val = chunk->accessor->value(v)` [S1, unofficial].
7. **Insert or Reject:**
   - If `val` exists:
     - If `val->num_points < max_points_per_voxel`:
       - Append $p$ to `val->points[val->num_points]`.
       - Increment `val->num_points`.
       - Set `chunk->state = MODIFIED`.
       - Increment `insert_count` [S1, unofficial].
     - Else:
       - Increment `rejected_count` [S1, unofficial].
   - If `val` is null:
     - Create a new `CellT` cell.
     - Set `cell.points[0] = p`.
     - Set `cell.num_points = 1`.
     - Call `chunk->accessor->setValue(v, cell)`.
     - Set `chunk->state = MODIFIED`.
     - Increment `insert_count` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_voxelization_with_capacity_limits(points, resolution, max_points_per_voxel, current_source_chunk_idx, chunk_map, sliding_window_size):
    inv_resolution = 1.0 / resolution
    insert_count = 0
    out_of_range_count = 0
    rejected_count = 0
    
    for p in points:
        # Step 1: Validate Point
        if not is_finite(p.x) or not is_finite(p.y) or not is_finite(p.z):
            continue
            
        # Step 2: Compute Voxel Coordinate
        voxel_coord = floor(p * inv_resolution)
        
        # Step 3: Determine Chunk Index
        point_chunk_idx = get_corresponding_chunk_idx(voxel_coord)
        
        # Step 4: Check Window
        if not is_neighbor_chunk(current_source_chunk_idx, point_chunk_idx):
            out_of_range_count += 1
            continue
            
        # Step 5: Wait for Chunk
        point_chunk = chunk_map[point_chunk_idx]
        wait_start = now()
        while not point_chunk.ready:
            sleep(1ms)
            if now() - wait_start > 10ms:
                log_error("ERROR: chunk load timeout")
                return False, insert_count, rejected_count, out_of_range_count
                
        # Step 6: Access Voxel
        val = point_chunk.accessor.value(voxel_coord)
        
        # Step 7: Insert or Reject
        if val is not None:
            if val.num_points < max_points_per_voxel:
                val.points[val.num_points] = p
                val.num_points += 1
                point_chunk.state = "MODIFIED"
                insert_count += 1
            else:
                rejected_count += 1
        else:
            cell = CellT()
            cell.points[0] = p
            cell.num_points = 1
            point_chunk.accessor.set_value(voxel_coord, cell)
            point_chunk.state = "MODIFIED"
            insert_count += 1
            
    return True, insert_count, rejected_count, out_of_range_count
```

**Parameters:**
- `max_points_per_voxel`: Typical value 10 [S1, unofficial].
- `resolution`: Typical value 0.2 [S1, unofficial].

**Pitfalls:**
- Chunk readiness must be polled with a timeout to avoid infinite blocking during disk I/O operations [S1, unofficial].

## Subtopics

### [Probabilistic Raytracing Occupancy Update](12215-probabilistic-raytracing-occupancy-update.md)
Updates voxel occupancy probabilities using log-odds from raytracing hits/misses, differing by introducing probabilistic confidence rather than hard capacity limits.

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ points, assuming $O(1)$ hash map access. Chunk waiting adds latency dependent on I/O speed.
- **Memory Complexity:** Proportional to the number of occupied voxels, each storing up to `max_points_per_voxel` points.
- **Strengths:** Handles asynchronous data loading via chunk readiness polling; prevents memory overflow per voxel.
- **Limitations:** Sequential point processing in the provided implementation; potential for high rejection rates if `max_points_per_voxel` is too low for dense regions.
- **When to use:** Streaming point cloud ingestion where chunk-based memory management is required.
- **When not to use:** When deterministic ordering of points within a voxel is critical without capacity limits, or when GPU acceleration is needed (see [Fast Deterministic GPU Voxelization (Sort-Scatter)](3806-fast-deterministic-gpu-voxelization-sort-scatter.md)).

## Background
Originated in the `voxlvoxl` project by goguma426, this method extends deterministic hard voxelization to support sliding window operations and disk-backed sparse voxel grids [S1, unofficial].

## Sources
[S1] goguma426/voxlvoxl, https://github.com/goguma426/voxlvoxl, fetched 2026-10-28 (unofficial)
