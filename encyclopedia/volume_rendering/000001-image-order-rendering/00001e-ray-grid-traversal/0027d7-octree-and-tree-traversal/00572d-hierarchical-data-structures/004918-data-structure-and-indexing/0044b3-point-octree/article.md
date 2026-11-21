# Point Octree


## Overview
A Point Octree is a recursive hierarchical data structure designed to manage a dynamic set of 3D points, supporting insertion, removal, movement, and spatial queries such as nearest neighbor and radius search [S1, unofficial]. It differs from voxel-based octrees by storing data only at leaf nodes corresponding to occupied regions, allowing for efficient handling of sparse point clouds without allocating memory for empty space [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D points $\mathbf{p}_i$ with associated user data $D_i$.
*   **Output:** A tree structure enabling $O(\log N)$ average-case spatial queries and updates.

**Data Structures:**
The tree consists of nodes (octants). Each node contains:
*   `children`: An array of up to 8 child octants (or `null` if leaf).
*   `data`: A `PointData<T>` object (or `null` if internal node).
*   `PointData<T>` layout: Two parallel arrays:
    *   `points: Vector3[]`: Stores the coordinates of points in this leaf.
    *   `data: T[]`: Stores the associated user data for each point.

**Parameters:**
*   `maxPoints`: Maximum number of points allowed in a leaf before splitting (default not specified in sources) [S1, unofficial].
*   `maxDepth`: Hard limit on recursion depth to prevent infinite subdivision [S1, unofficial].
*   `bias`: A padding value added to octant boundaries during containment checks to handle points near boundaries or sphere intersections [S1, unofficial].

**Algorithm Steps:**

1.  **Insertion (`set`):**
    *   Recursively traverse from the root to find the leaf octant containing the point $\mathbf{p}$.
    *   Check if $\mathbf{p}$ already exists in the leaf's `points` array. If yes, update the corresponding entry in `data` [S1, unofficial].
    *   If the leaf is full (number of points $\ge$ `maxPoints`) and current depth < `maxDepth`, call `split()` and `redistribute()` [S1, unofficial].
    *   Otherwise, append $\mathbf{p}$ to `points` and $D$ to `data` in the leaf [S1, unofficial].

2.  **Splitting and Redistribution:**
    *   `split()`: Create 8 empty child octants.
    *   `redistribute(bias)`: Iterate through all points in the current node's `data`. For each point, determine which child octant contains it (using `contains(point, bias)`). Move the point and its data to the child's `PointData` [S1, unofficial].
    *   Set the current node's `data` to `null` [S1, unofficial].

3.  **Radius Search (`findPoints`):**
    *   Recursively traverse child octants that intersect the query sphere (defined by center $\mathbf{c}$ and radius $r$). Intersection is checked using `contains(center, r)` [S1, unofficial].
    *   In leaf nodes, compute squared distance $d^2 = \|\mathbf{p}_i - \mathbf{c}\|^2$ for each point.
    *   Filter points where $d^2 \le r^2$ [S1, unofficial].

4.  **Movement (`move`):**
    *   If the point and new position are in the same octant, update the position in place [S1, unofficial].
    *   If they diverge, remove the point from the old location and insert it at the new location [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class PointData:
    def __init__(self):
        self.points = []  # List of Vector3
        self.data = []    # List of T

class PointOctant:
    def __init__(self, min_bound, max_bound):
        self.min = min_bound
        self.max = max_bound
        self.children = None  # List of PointOctant or None
        self.data = None      # PointData or None

    def contains(self, point, bias=0):
        """Check if point is inside octant boundaries with optional bias."""
        return (point.x >= self.min.x - bias and point.x <= self.max.x + bias and
                point.y >= self.min.y - bias and point.y <= self.max.y + bias and
                point.z >= self.min.z - bias and point.z <= self.max.z + bias)

    def split(self):
        """Create 8 children."""
        mid = (self.min + self.max) / 2
        self.children = []
        for i in range(8):
            # Determine child bounds based on bit index i
            # x: (i>>2)&1, y: (i>>1)&1, z: i&1
            x_bit = (i >> 2) & 1
            y_bit = (i >> 1) & 1
            z_bit = i & 1
            
            new_min = Vector3(
                self.min.x if x_bit == 0 else mid.x,
                self.min.y if y_bit == 0 else mid.y,
                self.min.z if z_bit == 0 else mid.z
            )
            new_max = Vector3(
                self.max.x if x_bit == 0 else mid.x,
                self.max.y if y_bit == 0 else mid.y,
                self.max.z if z_bit == 0 else mid.z
            )
            self.children.append(PointOctant(new_min, new_max))
        self.data = None

    def redistribute(self, bias):
        """Move points to children."""
        if self.data is None or self.children is None:
            return
        for i in range(len(self.data.points)):
            p = self.data.points[i]
            d = self.data.data[i]
            for child in self.children:
                if child.contains(p, bias):
                    if child.data is None:
                        child.data = PointData()
                    child.data.points.append(p)
                    child.data.data.append(d)
                    break
        self.data = None

    def set(self, point, data_entry, max_points, max_depth, current_depth, bias):
        """Insert or update a point."""
        if current_depth >= max_depth:
            # Hard limit reached, append anyway or handle error
            if self.data is None:
                self.data = PointData()
            self.data.points.append(point)
            self.data.data.append(data_entry)
            return

        if self.children is None:
            # Leaf node
            if self.data is None:
                self.data = PointData()
            
            # Check for existing point
            for i in range(len(self.data.points)):
                if self.data.points[i].equals(point):
                    self.data.data[i] = data_entry
                    return
            
            # If full, split and redistribute
            if len(self.data.points) >= max_points:
                self.split()
                self.redistribute(bias)
                # Now recurse into children
                for child in self.children:
                    if child.contains(point, bias):
                        child.set(point, data_entry, max_points, max_depth, current_depth + 1, bias)
                        return
            else:
                self.data.points.append(point)
                self.data.data.append(data_entry)
        else:
            # Internal node, recurse
            for child in self.children:
                if child.contains(point, bias):
                    child.set(point, data_entry, max_points, max_depth, current_depth + 1, bias)
                    return

    def find_points(self, center, radius, bias, results):
        """Find points within radius."""
        if self.data is None:
            return
        # Check if octant intersects sphere
        if not self.contains(center, radius):
            return
        
        if self.children is None:
            # Leaf node
            for i in range(len(self.data.points)):
                d_sq = self.data.points[i].distance_squared(center)
                if d_sq <= radius * radius:
                    results.append((self.data.points[i], self.data.data[i]))
        else:
            # Internal node
            for child in self.children:
                child.find_points(center, radius, bias, results)
```

## Subtopics

### Octree-Based Spatial Query Acceleration
Uses an octree (or kDTree) to accelerate spatial queries like nearest neighbor and radius search, avoiding $O(N^2)$ brute-force comparisons [S2, unofficial]. On a 1.42 million point indoor scan, building a cKDTree index took 417 ms [S2, unofficial]. For 300k points, 16-Nearest Neighbor queries took 4.1 ms with the tree vs 5,071 ms brute force (1,224x speedup) [S2, unofficial]. Radius search (r=0.15m) took 26 ms vs 3,246 ms (126x speedup) [S2, unofficial]. Brute force is faster for single lookups, but the tree amortizes build cost over many queries [S2, unofficial]. Used for normal estimation, segmentation, and ground filtering [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Average case for insertion, deletion, and spatial queries is $O(\log N)$ [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of points and the tree depth. No memory is allocated for empty space [S1, unofficial].
*   **Performance:** On a 1.42M point scan, index build took 417 ms [S2, unofficial]. 16-NN query on 300k points: 4.1 ms (tree) vs 5,071 ms (brute force) [S2, unofficial].
*   **Strengths:** Efficient for sparse data; supports dynamic updates (insert/remove/move) [S1, unofficial].
*   **Limitations:** `maxDepth` is a hard limit that may prevent perfect balancing for very dense clusters [S1, unofficial]. Build cost is non-trivial for large static datasets if queries are few [S2, unofficial].

## Background
The Point Octree is a standard recursive octree implementation for managing 3D points [S1, unofficial]. It is widely used in point cloud processing for tasks requiring spatial indexing, such as normal estimation and segmentation [S2, unofficial]. The specific implementation details (parallel arrays, bias-based containment) are derived from the `sparse-octree` library [S1, unofficial].

## Sources
[S1] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
[S2] Point Cloud Level of Detail: Octrees and Out-of-Core ..., https://learngeodata.eu/point-cloud-level-of-detail-octree/, fetched 2026-10-28 (unofficial)
