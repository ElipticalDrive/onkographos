# Octree Construction with Dynamic Depth Stop Criterion

Also called: Dynamic octree subdivision, Point cloud octree construction

## Overview
This method constructs a uniform-depth octree from a point cloud by dynamically determining the subdivision depth based on an occupancy ratio threshold, rather than using a fixed maximum depth or a simple point-count limit [S1, unofficial]. It solves the problem of inconsistent voxel sizes and excessive memory usage caused by isolated points or arbitrary depth limits, ensuring all leaf nodes reside at the same level for compact storage [S1, unofficial]. It differs from fixed-depth approaches by adapting to the geometric density of each pose, and from occupancy-count limits by preventing isolated points from creating disproportionately large octants [S1, unofficial].

## How it works
**Inputs:**
- A set of 3D points $P = \{p_1, ..., p_n\}$ representing the point cloud.
- A threshold parameter $\delta$ (typically $0.05$ or $5\%$) [S1, unofficial].

**Outputs:**
- An octree where all leaf nodes are at the same depth $h$.
- Each leaf node contains a subset of points.

**Data Structures:**
- **Octree Node:** Contains an 8-bit occupancy mask indicating which of the 8 child octants are non-empty. Since all leaves are at the same level, the tree can be stored in a breadth-first array using 1 byte per node [S1, unofficial].
- **Point List:** Each leaf node stores the list of points contained within its bounding volume.

**Steps:**

1.  **Compute Global Bounding Box:**
    Calculate the Axis-Aligned Bounding Box (AABB) of the entire point cloud $P$. Let $L_x, L_y, L_z$ be the lengths of the box in each dimension. Determine the maximum length $L_{max} = \max(L_x, L_y, L_z)$ [S1, unofficial].

2.  **Normalize to Cube:**
    Scale the AABB so that it becomes a cube with edge length $L_{max}$. This ensures that all resulting voxels are cubic rather than rectangular prisms [S1, unofficial].

3.  **Initialize Root:**
    Create the root node of the octree. Assign all points $P$ to the root.

4.  **Recursive Subdivision:**
    For a given node at level $h$ containing a set of points $P_{node}$:
    a.  **Classify Points:** Determine which of the 8 octants contain at least one point from $P_{node}$.
    b.  **Create Children:** For each non-empty octant, create a child node and assign the corresponding points to it.
    c.  **Check Stop Criterion:** Let $oct(h)$ be the total number of occupied octants at level $h$ across the entire tree. Let $n$ be the total number of points in the original cloud $P$.
        The subdivision stops for a specific branch if the global condition is met:
        $$ \frac{oct(h)}{n} < 1 - \delta $$
        If this condition is true, no further subdivision occurs for any node at level $h$ or deeper. All current nodes at level $h$ become leaves [S1, unofficial].
    d.  **Recurse:** If the condition is not met, recurse on each child node.

5.  **Leaf Assignment:**
    Points are stored in the leaf nodes. The number of points per leaf varies, but the spatial extent (voxel size) is consistent for all leaves [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_dynamic_octree(points, delta):
    """
    points: list of 3D vectors
    delta: float threshold (e.g., 0.05)
    Returns: list of leaf nodes in breadth-first order
    """
    if not points:
        return []

    # Step 1 & 2: Compute bounding box and normalize
    min_coord = min(p.min() for p in points) # Element-wise min
    max_coord = max(p.max() for p in points) # Element-wise max
    
    # Calculate dimensions
    dims = [max_coord[i] - min_coord[i] for i in range(3)]
    max_dim = max(dims)
    
    # Scale factor to make it a cube
    # Note: In implementation, this usually involves transforming coordinates
    # or adjusting the voxel size calculation. Here we assume coordinates
    # are transformed to fit a cube of side max_dim.
    scale_factors = [max_dim / d if d > 0 else 1.0 for d in dims]
    
    # Transform points to normalized cube space [0, max_dim]^3
    normalized_points = []
    for p in points:
        new_p = [(p[i] - min_coord[i]) * scale_factors[i] for i in range(3)]
        normalized_points.append(new_p)
        
    n = len(normalized_points)
    
    # Step 3: Initialize root
    root = Node(bounding_box=[0, max_dim], points=normalized_points)
    
    # We need to track the total number of occupied octants at each level
    # to check the global stop criterion.
    
    current_level_nodes = [root]
    level = 0
    
    while True:
        level += 1
        next_level_nodes = []
        total_occupied_octants = 0
        
        for node in current_level_nodes:
            # Step 4a: Classify points into 8 octants
            octants = [[] for _ in range(8)]
            for p in node.points:
                octant_idx = get_octant_index(p, node.bounding_box)
                octants[octant_idx].append(p)
                
            # Step 4b: Create children for non-empty octants
            for i, pts in enumerate(octants):
                if pts:
                    total_occupied_octants += 1
                    child_box = get_octant_box(node.bounding_box, i)
                    child = Node(bounding_box=child_box, points=pts)
                    next_level_nodes.append(child)
                    
        # Step 4c: Check stop criterion
        # Condition: oct(h)/n < 1 - delta
        # If true, we stop subdividing. The current 'next_level_nodes' 
        # are the leaves.
        if total_occupied_octants / n < 1 - delta:
            return next_level_nodes
            
        current_level_nodes = next_level_nodes

def get_octant_index(point, box):
    """
    Determines which of the 8 octants a point falls into.
    box: [min_x, max_x, min_y, max_y, min_z, max_z]
    """
    mid_x = (box[0] + box[1]) / 2
    mid_y = (box[2] + box[3]) / 2
    mid_z = (box[4] + box[5]) / 2
    
    idx = 0
    if point[0] >= mid_x: idx |= 1
    if point[1] >= mid_y: idx |= 2
    if point[2] >= mid_z: idx |= 4
    return idx

def get_octant_box(box, idx):
    """
    Returns the bounding box for a specific octant index.
    """
    min_x, max_x, min_y, max_y, min_z, max_z = box
    mid_x = (min_x + max_x) / 2
    mid_y = (min_y + max_y) / 2
    mid_z = (min_z + max_z) / 2
    
    new_min = [min_x, min_y, min_z]
    new_max = [max_x, max_y, max_z]
    
    if idx & 1: new_min[0] = mid_x
    else: new_max[0] = mid_x
    
    if idx & 2: new_min[1] = mid_y
    else: new_max[1] = mid_y
    
    if idx & 4: new_min[2] = mid_z
    else: new_max[2] = mid_z
    
    return new_min + new_max

class Node:
    def __init__(self, bounding_box, points):
        self.bounding_box = bounding_box
        self.points = points
```

**Parameters:**
- $\delta$: Threshold for acceptable loss. Typical value: $0.05$ ($5\%$) [S1, unofficial].
  - $\delta = 0$: Ideal 1:1 ratio between terminal octants and points. Causes aggressive subdivision (dozens/hundreds of levels), precision problems, and excessive node allocation [S1, unofficial].
  - $\delta = 0.5$: Coarser hierarchy, 50% loss allowed [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Fixed Depth:** Using a fixed maximum depth is tedious as different poses of 4D media require different depths [S1, unofficial].
- **Occupancy Count Limit:** Stopping when a node contains fewer than $k$ points causes isolated points to occupy large octants, creating incoherent voxel sizes [S1, unofficial].
- **Precision:** Extreme subdivision levels (e.g., $\delta=0$) lead to numeric precision issues and unnecessary memory usage without rendering benefits, as LOD traversal halts at coarser levels based on screen projection [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the depth $h$ determined by $\delta$. In the worst case ($\delta=0$), depth can be very large, leading to high computational cost. In practice, with $\delta=0.05$, the depth is moderate [S1, unofficial].
- **Memory Complexity:** All leaf nodes are at the same level, allowing compact breadth-first storage with 1 byte per node (8-bit mask) [S1, unofficial]. This is more efficient than general octrees with varying depths.
- **Performance:** The method adapts to the geometry, avoiding over-subdivision in sparse areas and under-subdivision in dense areas [S1, unofficial].
- **Strengths:** Automatic depth adjustment, consistent voxel size, compact storage, handles 4D media poses effectively [S1, unofficial].
- **Limitations:** Requires global knowledge of $n$ (total points) to evaluate the stop criterion, which may necessitate a two-pass algorithm or pre-computation [S1, unofficial].

## Background
This method was proposed in the context of hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4D media [S1, unofficial]. It addresses the limitations of previous octree construction methods for dynamic point clouds, where fixed depths or simple occupancy counts failed to provide consistent and efficient representations [S1, unofficial].

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
