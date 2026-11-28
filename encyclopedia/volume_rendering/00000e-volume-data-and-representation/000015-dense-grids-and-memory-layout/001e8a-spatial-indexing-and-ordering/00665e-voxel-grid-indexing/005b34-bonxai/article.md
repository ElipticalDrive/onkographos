# Bonxai

Also called: BonxaiGrid

## Overview
Bonxai is a header-only C++ sparse voxel grid library designed to represent 3D space, specifically intended to replace simple hash table maps in LiDAR odometry pipelines [S1, unofficial]. It addresses the performance bottlenecks of nearest neighbor searches and point removal operations found in previous implementations like `VoxelHashMap` [S1, unofficial]. Unlike flat hash tables, Bonxai uses a hierarchical structure with dense leaf nodes, which significantly accelerates query operations at the cost of increased memory consumption proportional to data sparsity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D point cloud data (coordinates).
*   **Output:** A hierarchical grid structure supporting insertion, nearest neighbor search, and point removal.

**Data Structures:**
The grid is hierarchical, consisting of inner grids and leaf nodes [S1, unofficial].
*   **Inner Grid:** A sparse index layer that points to active leaf nodes.
*   **Leaf Node:** A dense allocation of voxels. The dimensions are controlled by `leaf_grid_log2_dim`.
    *   Number of voxels per leaf: $2^{\text{leaf\_grid\_log2\_dim} \times 3}$ [S1, unofficial].
    *   Default `leaf_grid_log2_dim` is 3, resulting in $2^9 = 512$ voxels per leaf [S1, unofficial].
    *   Alternative "small leaf" configuration uses `leaf_grid_log2_dim` = 2, resulting in $2^6 = 64$ voxels per leaf [S1, unofficial].
*   **Voxel Content:** Each voxel stores a `std::vector` of points [S1, unofficial].

**Parameters:**
*   `inner_grid_log2_dim`: Log2 of the inner grid dimension. Default: 2 [S1, unofficial].
*   `leaf_grid_log2_dim`: Log2 of the leaf grid dimension. Default: 3 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Create the grid with specified `inner_grid_log2_dim` and `leaf_grid_log2_dim`.
2.  **Insertion:**
    a. Map the input 3D point to its corresponding leaf node and local voxel index within that leaf.
    b. If the leaf node does not exist, allocate it.
    c. Access the `std::vector` of points in the target voxel.
    d. Reserve memory for the vector if it is empty or small, using `max_points_per_voxel_` [S1, unofficial].
    e. Append the point to the vector.
3.  **Nearest Neighbor Search:**
    a. Identify the leaf node containing the query point.
    b. Iterate through the voxels in the vicinity (typically the same leaf or adjacent leaves depending on implementation details not fully specified in sources, but implied by the hierarchical nature) [S1, unofficial].
    c. Compute distances to points stored in the voxel vectors.
    d. Return the closest point.
4.  **Point Removal:**
    a. Locate the voxel containing the point.
    b. Remove the point from the `std::vector`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BonxaiGrid:
    def __init__(self, inner_grid_log2_dim=2, leaf_grid_log2_dim=3):
        self.inner_grid_log2_dim = inner_grid_log2_dim
        self.leaf_grid_log2_dim = leaf_grid_log2_dim
        self.leaf_size = 2 ** leaf_grid_log2_dim
        self.inner_size = 2 ** inner_grid_log2_dim
        # Inner grid is a map from inner coordinate to LeafNode
        self.inner_grid = {}
        self.max_points_per_voxel = 100 # Example default, not explicitly given in sources [G]

    class LeafNode:
        def __init__(self):
            # Dense array of vectors for voxels
            num_voxels = 2 ** (self.leaf_grid_log2_dim * 3)
            self.voxels = [VectorOfPoints() for _ in range(num_voxels)]

    class VectorOfPoints:
        def __init__(self):
            self.points = []
        
        def reserve(self, n):
            # Pre-allocate memory for n points
            # In C++ std::vector, this ensures capacity >= n
            pass # Implementation detail of std::vector [G]

    def insert_point(self, point):
        # 1. Determine leaf and voxel index
        # This requires coordinate space conversion logic defined in parent articles
        leaf_coord = get_leaf_coordinate(point, self.leaf_size)
        voxel_local_index = get_voxel_local_index(point, self.leaf_size)
        
        # 2. Get or create leaf node
        if leaf_coord not in self.inner_grid:
            self.inner_grid[leaf_coord] = LeafNode()
        
        leaf = self.inner_grid[leaf_coord]
        
        # 3. Access voxel and reserve if needed
        voxel_vec = leaf.voxels[voxel_local_index]
        if len(voxel_vec.points) == 0:
            voxel_vec.reserve(self.max_points_per_voxel)
            
        # 4. Add point
        voxel_vec.points.append(point)

    def nearest_neighbor(self, query_point):
        # 1. Determine leaf coordinate
        leaf_coord = get_leaf_coordinate(query_point, self.leaf_size)
        
        if leaf_coord not in self.inner_grid:
            return None # No points in this region
            
        leaf = self.inner_grid[leaf_coord]
        
        # 2. Search within leaf
        # Determine relevant voxels based on query point's local position
        # For simplicity, assume search within the same leaf for now
        # In practice, may need to check adjacent leaves if query is near boundary
        
        min_dist = infinity
        closest_point = None
        
        # Iterate over voxels in the leaf
        # Optimization: only check voxels within search radius
        for voxel_idx in get_voxels_in_radius(query_point, leaf.leaf_grid_log2_dim):
            voxel_vec = leaf.voxels[voxel_idx]
            for point in voxel_vec.points:
                dist = distance(query_point, point)
                if dist < min_dist:
                    min_dist = dist
                    closest_point = point
                    
        return closest_point
```

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** Dense leaf allocation causes memory overhead proportional to data sparsity. If a leaf node is mostly empty, the memory for the `std::vector` array is still allocated [S1, unofficial].
*   **Performance Degradation:** Performance degrades in scenarios with random point measurements (reflections, occlusions) or sparse structures, as the dense leaf structure is less efficient than hash tables for highly sparse data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Nearest neighbor search and point removal are faster than `VoxelHashMap` [S1, unofficial]. Point insertion is slower than `VoxelHashMap` [S1, unofficial].
*   **Memory Complexity:** Higher memory consumption than hash tables due to dense leaf allocation. The overhead is proportional to the sparsity of the data [S1, unofficial].
*   **Measured Performance:**
    *   On a benchmark of 6,000,000 points sampled from a 100m cube surface, Bonxai halves the runtime of the nearest neighbor search compared to `VoxelHashMap` [S1, unofficial].
    *   On 60,000 points, memory and runtime trends remain consistent [S1, unofficial].
    *   The speedup is particularly notable for single-threaded nearest neighbor searches [S1, unofficial].
*   **Strengths:** Fast nearest neighbor queries, fast point removal, header-only, no additional dependencies, ROS ecosystem integration [S1, unofficial].
*   **Limitations:** Higher memory usage, slower point insertion, performance issues with highly sparse or random data [S1, unofficial].
*   **When to use:** When nearest neighbor search is a bottleneck and memory is not the primary constraint, especially in structured environments [S1, unofficial].
*   **When not to use:** When memory is extremely constrained or data is highly sparse and random [S1, unofficial].

## Background
Bonxai was introduced as a replacement for the `VoxelHashMap` in the KISS-ICP project by the PRBonn group [S1, unofficial]. The motivation was to move away from simple hash tables to a more appropriate and flexible data structure for representing 3D space, enabling future extensions like TSDF registration [S1, unofficial]. The library was chosen for being well-maintained, header-only, and integrated with the ROS ecosystem [S1, unofficial]. Benchmarks were conducted using `heaptrack` for memory analysis and comparing against `VoxelHashMap` [S1, unofficial].

## Sources
[S1] Introduce Bonxai as Map representation, https://github.com/PRBonn/kinematic-icp/pull/22, fetched 2026-10-27 (unofficial)
