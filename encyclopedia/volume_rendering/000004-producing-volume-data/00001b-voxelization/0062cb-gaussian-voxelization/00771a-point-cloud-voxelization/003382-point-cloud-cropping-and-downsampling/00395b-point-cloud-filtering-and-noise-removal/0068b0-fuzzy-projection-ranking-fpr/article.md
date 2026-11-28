# Fuzzy Projection Ranking (FPR)

Also called: FPR (Fast Point Cloud Filtering by Rank), Fast Point Ranking

## Overview
Fuzzy Projection Ranking (FPR) is a point cloud processing technique that projects 3D LiDAR points onto a 2D range image grid to compute a fuzzy rank for each point based on the intensity (range) distribution of its neighbors, thereby filtering noise and enhancing features [S1, unofficial]. It is designed for robust cloud voxelization and denoising in LiDAR odometry and mapping, particularly in adverse weather conditions such as rain, fog, and dust [S1, unofficial]. Unlike sibling methods like Statistical Outlier Removal which rely on global or local statistical distributions in 3D space, FPR leverages the structured nature of LiDAR scans via 2D projection and rank-based voxelization to preserve geometric structure while removing outliers [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A raw LiDAR point cloud (`cloud`) containing points with fields $x, y, z$, intensity, timestamp, and ring index.
- **Outputs:**
  - `cloud_trimmed`: Points within valid range bounds.
  - `cloud_ranked`: Points with an added rank field.
  - `cloud_voxelized`: Voxelized point cloud.
  - `filtered`: Final denoised point cloud.
  - `Neighbours`: A 2D matrix mapping range image coordinates to point indices.

**Data Structures:**
- **PointType:** Struct containing $x, y, z$ (float), intensity (float), timestamp (double), ring (uint16_t) [S1, unofficial].
- **VoxelAccum:** Struct containing `weighted_sum` (Eigen::Vector3d), `weight_sum` (double, initialized to 0.0001), `max_intensity` (float, initialized to -1.0f), and `max_point` (PointType) [S1, unofficial].
- **VoxelHash:** Hash function for voxel indices $(i, j, k)$ computed as $((1 \ll 20) - 1) \& (i \cdot 73856093 \oplus j \cdot 19349663 \oplus k \cdot 83492791)$ [S1, unofficial].
- **Neighbours:** Matrix of size $N\_SCAN \times Horizon\_SCAN$ storing the index of the point projected to each pixel, initialized to -1 [S1, unofficial].

**Parameters:**
- $N\_SCAN$: Number of vertical channels (rings). Typical values: 32 (Hesai) or 64 (HDL-64) [S1, unofficial].
- $ang\_res\_x$: Horizontal angular resolution in degrees. Typical values: 0.18 (Hesai) or 0.2 (HDL-64) [S1, unofficial].
- $Horizon\_SCAN$: Number of horizontal columns, calculated as $360 / ang\_res\_x$. Typical value: 2000 [S1, unofficial].
- $max\_range$: Maximum valid range for point acceptance. Typical value: 100.0 (or 30.0 in some implementations) [S1, unofficial].
- $kernel\_row$: Half-height of the 2D convolution kernel. Value: 1 [S1, unofficial].
- $kernel\_col$: Half-width of the 2D convolution kernel. Value: 5 [S1, unofficial].
- $voxel\_size$: Size of the voxel grid in meters. Typical value: 0.5 [S1, unofficial].
- $drop\_percent$: Percentage of lowest-rank points to drop. Typical value: 10 [S1, unofficial].
- $use\_weighted$: Boolean flag for voxelization mode (weighted centroid vs. max intensity) [S1, unofficial].

**Steps:**

1. **Range Image Projection:**
   For each point $p$ in `cloud`:
   - Calculate range $r = \sqrt{x^2 + y^2 + z^2}$.
   - If $r < 0.1$ or $r > max\_range$, skip the point [S1, unofficial].
   - Calculate row: $row = ring$ [S1, unofficial].
   - Calculate horizontal angle: $horizonAngle = \text{atan2}(x, y) \cdot 180 / \pi$.
   - Calculate column: $col = -\text{round}((horizonAngle - 90.0) / ang\_res\_x) + Horizon\_SCAN / 2$.
   - Handle horizontal wrapping:
     - If $col \ge Horizon\_SCAN$, set $col = col - Horizon\_SCAN$.
     - If $col < 0$ or $col \ge Horizon\_SCAN$, skip the point [S1, unofficial].
   - Store the point index in `Neighbours[row, col]`.
   - Add the point to `cloud_trimmed` with its range stored in the intensity field for subsequent rank calculation [S1, unofficial].

2. **Rank Estimation:**
   For each point $p$ in `cloud_trimmed`:
   - Retrieve its projected coordinates $(row, col)$ and current range $range\_curr = p.intensity$.
   - Initialize $rank = 0.0$ and $count = 0$.
   - Iterate over neighbors defined by the kernel offsets $(dy, dx)$ where $dy \in [-kernel\_row, kernel\_row]$ and $dx \in [-kernel\_col, kernel\_col]$, excluding $(0,0)$ [S1, unofficial].
   - For each neighbor offset:
     - Calculate neighbor coordinates: $y\_ = row + dy$, $x\_ = col + dx$.
     - Handle vertical boundary: If $y\_ < 0$ or $y\_ \ge N\_SCAN$, continue [S1, unofficial].
     - Handle horizontal wrapping:
       - If $x\_ < 0$, set $x\_ = Horizon\_SCAN - 1 + x\_$.
       - If $x\_ \ge Horizon\_SCAN$, set $x\_ = x\_ - Horizon\_SCAN - 1$.
       - If $x\_ < 0$ or $x\_ \ge Horizon\_SCAN$, continue [S1, unofficial].
     - Retrieve neighbor index $idx\_n = Neighbours[y\_, x\_].
     - If $idx\_n > 0$:
       - Get neighbor point $p\_n = cloud[idx\_n]$.
       - Calculate neighbor range $r\_neigh = \sqrt{p\_n.x^2 + p\_n.y^2 + p\_n.z^2}$.
       - Calculate difference $diff = r\_neigh - range\_curr$.
       - Update rank: $rank += \exp(-0.5 \cdot diff^2)$.
       - Increment $count$.
   - If $count > 0$, normalize rank: $rank = rank / count$.
   - Calculate weight: $weight = range\_curr / max\_range$.
   - Finalize rank: $rank = (1 + rank) \cdot (1 + weight)$.
   - Store the rank in $p.intensity$ [S1, unofficial].

3. **Intensity Filtering:**
   - Calculate number of points to drop: $num\_to\_drop = \text{int}((drop\_percent / 100.0) \cdot len(cloud\_ranked))$.
   - Sort points in `cloud_ranked` by rank (stored in intensity) in ascending order.
   - Remove the first $num\_to\_drop$ points (lowest rank) to produce `filtered` [S1, unofficial].

4. **Voxelization:**
   - Initialize a hash map `grid` mapping voxel indices to `VoxelAccum`.
   - For each point $p$ in `filtered`:
     - Calculate voxel index: $voxel = \lfloor p / voxel\_size \rfloor$ (integer division) [S1, unofficial].
     - Retrieve or create `acc` for this voxel.
     - If $use\_weighted$ is true:
       - $acc.weighted\_sum += p.intensity \cdot (x, y, z)$.
       - $acc.weight\_sum += p.intensity$.
     - Else (max intensity mode):
       - If $p.intensity > acc.max\_intensity$:
         - $acc.max\_intensity = p.intensity$.
         - $acc.max\_point = p$.
   - Construct output cloud `cloud_voxelized`:
     - If $use\_weighted$:
       - Compute centroid: $p = acc.weighted\_sum / acc.weight\_sum$.
       - Set output point coordinates to $p$ and intensity to $acc.max\_intensity$ [S1, unofficial].
     - Else:
       - Set output point to $acc.max\_point$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def project_to_range_image(cloud, N_SCAN, Horizon_SCAN, ang_res_x, max_range):
    """
    Projects 3D points to a 2D range image grid.
    Returns cloud_trimmed, Neighbours matrix.
    """
    Neighbours = np.full((N_SCAN, Horizon_SCAN), -1, dtype=np.int32)
    cloud_trimmed = []
    
    for idx, pt in enumerate(cloud):
        x, y, z, ring = pt.x, pt.y, pt.z, pt.ring
        r = math.sqrt(x**2 + y**2 + z**2)
        
        # Range clipping
        if r < 0.1 or r > max_range:
            continue
            
        row = int(ring)
        if row < 0 or row >= N_SCAN:
            continue
            
        horizon_angle = math.degrees(math.atan2(x, y))
        col = -round((horizon_angle - 90.0) / ang_res_x) + Horizon_SCAN // 2
        
        # Horizontal wrapping
        if col >= Horizon_SCAN:
            col -= Horizon_SCAN
        if col < 0 or col >= Horizon_SCAN:
            continue
            
        Neighbours[row, col] = idx
        # Store range in intensity for rank calculation
        pt.intensity = r
        cloud_trimmed.append(pt)
        
    return cloud_trimmed, Neighbours

def compute_rank(cloud_trimmed, Neighbours, N_SCAN, Horizon_SCAN, max_range, kernel_row, kernel_col):
    """
    Computes fuzzy rank for each point based on neighbor range distribution.
    """
    # Generate kernel offsets
    kernel = []
    for dy in range(-kernel_row, kernel_row + 1):
        for dx in range(-kernel_col, kernel_col + 1):
            if dy == 0 and dx == 0:
                continue
            kernel.append((dy, dx))
            
    cloud_ranked = []
    
    for pt in cloud_trimmed:
        row = int(pt.ring) # Assuming ring is preserved or re-calculated if needed, 
                           # but projectToRangeImage sets row based on ring.
                           # Note: In the source, row is derived from pt.ring in projection.
                           # Here we assume pt has row/col or we re-project.
                           # The source code re-projects using projectPoint which uses pt.ring.
        
        # Re-project to get row/col if not stored
        horizon_angle = math.degrees(math.atan2(pt.x, pt.y))
        col = -round((horizon_angle - 90.0) / ang_res_x) + Horizon_SCAN // 2
        if col >= Horizon_SCAN:
            col -= Horizon_SCAN
            
        range_curr = pt.intensity
        rank = 0.0
        count = 0
        
        for dy, dx in kernel:
            y_ = row + dy
            x_ = col + dx
            
            # Vertical boundary check
            if y_ < 0 or y_ >= N_SCAN:
                continue
                
            # Horizontal wrapping
            if x_ < 0:
                x_ = Horizon_SCAN - 1 + x_
            if x_ >= Horizon_SCAN:
                x_ = x_ - Horizon_SCAN - 1
                
            if x_ < 0 or x_ >= Horizon_SCAN:
                continue
                
            idx_n = Neighbours[y_, x_]
            if idx_n > 0:
                p_n = cloud[idx_n] # Access original cloud
                r_neigh = math.sqrt(p_n.x**2 + p_n.y**2 + p_n.z**2)
                diff = r_neigh - range_curr
                rank += math.exp(-0.5 * diff * diff)
                count += 1
                
        if count > 0:
            rank /= count
            
        weight = range_curr / max_range
        rank = (1 + rank) * (1 + weight)
        
        pt.intensity = rank # Store rank in intensity
        cloud_ranked.append(pt)
        
    return cloud_ranked

def filter_by_rank_percent(cloud_ranked, drop_percent):
    """
    Drops the lowest rank percentage of points.
    """
    if not cloud_ranked or drop_percent <= 0:
        return cloud_ranked
        
    num_to_drop = int((drop_percent / 100.0) * len(cloud_ranked))
    
    # Sort by rank (stored in intensity)
    sorted_indices = np.argsort([p.intensity for p in cloud_ranked])
    kept_indices = sorted_indices[num_to_drop:]
    
    filtered = [cloud_ranked[i] for i in kept_indices]
    return filtered

def voxelize(filtered, voxel_size, use_weighted):
    """
    Voxelizes the filtered point cloud.
    """
    grid = {} # Hash map: voxel_index -> VoxelAccum
    
    for pt in filtered:
        # Calculate voxel index
        vx = int(pt.x / voxel_size)
        vy = int(pt.y / voxel_size)
        vz = int(pt.z / voxel_size)
        voxel = (vx, vy, vz)
        
        if voxel not in grid:
            grid[voxel] = {
                'weighted_sum': np.array([0.0, 0.0, 0.0]),
                'weight_sum': 0.0001,
                'max_intensity': -1.0,
                'max_point': None
            }
            
        acc = grid[voxel]
        
        if use_weighted:
            acc['weighted_sum'] += pt.intensity * np.array([pt.x, pt.y, pt.z])
            acc['weight_sum'] += pt.intensity
        else:
            if pt.intensity > acc['max_intensity']:
                acc['max_intensity'] = pt.intensity
                acc['max_point'] = pt
                
    cloud_voxelized = []
    for voxel, acc in grid.items():
        if use_weighted:
            centroid = acc['weighted_sum'] / acc['weight_sum']
            new_pt = PointType()
            new_pt.x, new_pt.y, new_pt.z = centroid
            new_pt.intensity = acc['max_intensity']
            cloud_voxelized.append(new_pt)
        else:
            if acc['max_point'] is not None:
                cloud_voxelized.append(acc['max_point'])
                
    return cloud_voxelized
```

## Subtopics

### Statistical Outlier Removal
A sibling method that removes outliers based on the statistical distribution of distances to neighbors in 3D space. It does not use 2D projection or rank-based weighting. Choose FPR when LiDAR scan structure is reliable and adverse weather noise is prevalent; choose Statistical Outlier Removal for general-purpose noise reduction where scan structure may be irregular [17819-statistical-outlier-removal.md].

## Operating characteristics
- **Time Complexity:** Projection is $O(N)$ where $N$ is the number of points. Rank computation is $O(N \cdot K)$ where $K$ is the number of kernel neighbors ($2 \cdot kernel\_row + 1 \times 2 \cdot kernel\_col + 1 - 1$). Voxelization is $O(N)$ on average using a hash map. Filtering is $O(N \log N)$ due to sorting.
- **Memory Complexity:** $O(N)$ for storing the point cloud and $O(N\_SCAN \cdot Horizon\_SCAN)$ for the `Neighbours` matrix.
- **Performance:** The method is parallelized using TBB for rank computation [S1, unofficial]. It is effective in adverse weather conditions (rain, fog, snow) [S1, unofficial].
- **Strengths:** Preserves geometric structure better than simple statistical filtering in noisy conditions; leverages LiDAR scan structure.
- **Limitations:** Requires accurate LiDAR parameters ($N\_SCAN$, $ang\_res\_x$); sensitive to horizontal wrapping errors; performance depends on kernel size.
- **When to use:** LiDAR odometry and mapping in adverse weather; when scan structure is consistent.
- **When not to use:** When LiDAR parameters are unknown or inaccurate; for non-LiDAR point clouds lacking ring structure.

## Background
Fuzzy Projection Ranking (FPR) was introduced by Eugeniu Vezeteu as an unofficial implementation for robust cloud voxelization and denoising in LiDAR odometry and mapping [S1, unofficial]. It is designed to handle adverse weather conditions such as rain, fog, and dust, which typically degrade LiDAR point cloud quality [S1, unofficial]. The method leverages the structured nature of LiDAR data by projecting points onto a 2D range image, computing a fuzzy rank based on neighbor intensity distributions, and then performing rank-based voxelization [S1, unofficial].

## Sources
[S1] eugeniu1994/FPR, https://github.com/eugeniu1994/FPR, fetched 2026-10-27 (unofficial)
