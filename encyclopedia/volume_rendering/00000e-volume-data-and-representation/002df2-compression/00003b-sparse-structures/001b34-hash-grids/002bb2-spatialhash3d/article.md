# SpatialHash3D
Also called: Uniform 3D Hash Grid
## Overview
SpatialHash3D is a spatial index structure using a uniform 3D hash grid to enable efficient nearest-neighbor and radius-based queries for static datasets, providing O(1) cell lookup [S1, unofficial]. It differs from its siblings by prioritizing implementation simplicity and predictable performance for static data over the adaptive granularity of octrees or the complex balancing of KD-trees, making it suitable for straightforward radius queries where data does not change during the query phase [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A static array of 3D points, where each point contains coordinates $(x, y, z)$ and an associated value.
- Output: A hash table mapping cell indices to lists of points, supporting queries for points within a radius or the $k$-nearest neighbors.

**Data Structures:**
- **Hash Table:** A dictionary or array mapping integer cell indices to a list of point references.
- **Cell Index:** An integer derived from the 3D coordinates.
- **Point Structure:** Contains `center: [x, y, z]` and associated data [S1, unofficial].

**Parameters:**
- `cellSize`: The size of each cell in the hash grid, determining the granularity of the spatial partition [S1, unofficial].
- `queryRadius`: The search radius for radius-based queries [S1, unofficial].
- `k`: The maximum number of nearest neighbors to return [S1, unofficial].

**Steps:**
1. **Build:** Iterate through all input points. For each point at $(x, y, z)$, calculate the cell index. Insert the point into the list associated with that cell index in the hash table.
2. **Query Radius:** Given a query point $(q_x, q_y, q_z)$ and a radius $r$:
   - Calculate the cell index of the query point.
   - Determine the range of neighboring cells that intersect with the sphere of radius $r$ centered at the query point.
   - Iterate through all points in these neighboring cells.
   - Calculate the Euclidean distance $d_i$ from the query point to each candidate point.
   - Return all points where $d_i \le r$.
3. **Query K-Nearest:** Similar to radius query, but maintain a priority queue or sorted list of the $k$ closest points found within the relevant cells, capped at $k$ [S1, unofficial].

**Formulas:**
- Cell Index Calculation: The specific hash function is not detailed, but it maps 3D coordinates to a unique integer index based on `cellSize` [S1, unofficial].
- Distance: $d_i = \sqrt{(x_i - q_x)^2 + (y_i - q_y)^2 + (z_i - q_z)^2}$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SpatialHash3D:
    def __init__(self, cell_size):
        self.cell_size = cell_size
        self.grid = {}  # Map from cell_index to list of points

    def _get_cell_index(self, x, y, z):
        # Calculate cell coordinates
        cx = int(x / self.cell_size)
        cy = int(y / self.cell_size)
        cz = int(z / self.cell_size)
        # Hash the cell coordinates to a single integer
        # Note: Specific hash function not provided in sources [G]
        return hash((cx, cy, cz))

    def build(self, points):
        for point in points:
            x, y, z = point['center']
            cell_idx = self._get_cell_index(x, y, z)
            if cell_idx not in self.grid:
                self.grid[cell_idx] = []
            self.grid[cell_idx].append(point)

    def query_radius(self, x, y, z, radius):
        results = []
        # Determine range of cells to check
        # Note: Exact neighbor cell calculation logic not provided in sources [G]
        # Iterate through relevant cells
        for cell_idx in self._get_neighbor_cells(x, y, z, radius):
            if cell_idx in self.grid:
                for point in self.grid[cell_idx]:
                    px, py, pz = point['center']
                    dist = ((px - x)**2 + (py - y)**2 + (pz - z)**2)**0.5
                    if dist <= radius:
                        results.append(point)
        return results

    def _get_neighbor_cells(self, x, y, z, radius):
        # Placeholder for neighbor cell iteration logic
        # This would calculate the bounding box of cells intersecting the sphere
        return [] # Not given in the sources.

    def query_k_nearest(self, x, y, z, k):
        # Similar to query_radius but limits results to k nearest
        # Note: Specific implementation details not provided in sources [G]
        return []
```

## Subtopics

### [Reprojected Spatial Hash Map Indices](5180-reprojected-spatial-hash-map-indices.md)
This method optimizes spatial hash lookups in temporal volume rendering by reusing hash indices from the previous frame, thereby reducing the frequency of expensive atomic memory operations on the hash grid. It differs from standard spatial hash lookups by decoupling the index computation from the final shading pass, using a texture-based history buffer to validate and propagate indices across frames.

## Operating characteristics
- **Time Complexity:** O(1) average case for cell lookup during build and query, assuming uniform distribution and efficient hashing [S1, unofficial]. Query time depends on the number of points in the relevant cells and the search radius.
- **Memory Complexity:** Proportional to the number of points and the number of occupied cells.
- **Performance:** Predictable performance for static datasets [S1, unofficial].
- **Strengths:** Simple implementation, easy radius queries, suitable for static data [S1, unofficial].
- **Limitations:** Not adaptive to data density; performance degrades if points are not uniformly distributed or if `cellSize` is poorly chosen.
- **When to use:** Static datasets where simplicity and predictable performance are prioritized [S1, unofficial].
- **When not to use:** Dynamic datasets requiring frequent updates or highly non-uniform data distributions where adaptive structures like octrees might be better.

## Background
SpatialHash3D is part of a pipeline for generating scalar fields from drill-hole sample data and rendering them as voxel volumes, specifically within the context of 3D IDW interpolation [S1, unofficial]. It was introduced as a recommended v1 implementation for spatial indexing in this context due to its straightforward nature and suitability for static data [S1, unofficial].

## Sources
[S1] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
