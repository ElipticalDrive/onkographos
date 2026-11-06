# Neighbor Counting


## Overview
Neighbor Counting estimates local point cloud density by counting the number of input points within a specified radius of each query point. It serves as a fundamental primitive for density evaluation in non-regular volumes, providing the scalar field data required by higher-level techniques like [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md) or clustering algorithms. It differs from [Chamfer Distance](2446-chamfer-distance.md) by measuring local neighborhood cardinality rather than pairwise distance errors, and from [Probabilistic Occupancy Policy](5043-probabilistic-occupancy-policy.md) by providing a deterministic count rather than a probabilistic occupancy estimate.

## How it works
**Inputs:**
- `points`: A set of $N$ 3D points, represented as an array of shape $(N, 3)$.
- `radius`: The search radius $r$ for neighbor identification.

**Outputs:**
- `counts`: An array of shape $(N,)$ where `counts[i]` is the number of neighbors of `points[i]` within distance `radius`.

**Data Structures:**
- **KDTree**: A spatial index structure built from `points` to enable efficient range queries. The exact internal layout (e.g., node type, split axis storage) is implementation-dependent but must support `query_radius` or equivalent functionality.

**Steps:**
1. **Build Index**: Construct a KDTree from the input `points` [S1, S3, unofficial].
2. **Query Neighbors**: For each point $p_i$ in `points`:
   - Query the KDTree for all points within distance `radius` of $p_i$ [S1, S3, unofficial].
   - Let $S_i$ be the set of returned indices.
   - Calculate the count: $c_i = |S_i| - 1$. The subtraction of 1 excludes the point $p_i$ itself, which is always within distance 0 [S1, unofficial].
3. **Store Result**: Store $c_i$ in the output array at index $i$.

**Pitfalls:**
- **Self-Inclusion**: Failing to subtract 1 from the query result leads to a count inflated by 1 for every point [S1, unofficial].
- **Boundary Effects**: In voxel-based variants, using a spherical query for cubic voxels yields imperfect boundary results, as the sphere extends beyond the cube faces [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def neighbor_counting(points: np.ndarray, radius: float) -> np.ndarray:
    """
    Calculate the number of neighbors for each point within a given radius.
    
    Args:
        points: Array of shape (N, 3) containing 3D coordinates.
        radius: Search radius for neighbors.
        
    Returns:
        Array of shape (N,) containing neighbor counts (excluding self).
    """
    # Step 1: Build KDTree
    tree = KDTree(points)
    
    # Step 2: Query neighbors
    # query_ball_point returns a list of lists, where each inner list contains
    # indices of points within the radius.
    neighbor_indices = tree.query_ball_point(points, r=radius)
    
    # Step 3: Calculate counts excluding self
    # The count is the length of the neighbor list minus 1 (for the point itself)
    counts = np.array([len(indices) - 1 for indices in neighbor_indices])
    
    return counts
```

## Subtopics

### Clusterability Check
Determines if a K-D tree node's grid can be clustered (optimized for rendering) by checking if the combined QEF (Quadratic Error Function) point count matches the sum of component point counts [S2, unofficial]. It calls `RectilinearGrid::calClusterability` with children grids, plane direction, min/max codes, and scalar field $t$. If `selfClusterable` is false, it sets `clusterable = false` and returns. It iterates over all children; if any child is not clusterable, it sets `clusterable = false` and returns. Otherwise, it sets `clusterable = true` [S2, unofficial].

### KDTree-based Neighbor Counting
Calculates the density of mesh vertices within each voxel by querying neighboring points using a KDTree and normalizing counts [S3, unofficial]. It uses `query_ball_point` to find all mesh vertices within a spherical radius around each voxel center. The sphere radius $r$ is calculated to match the voxel cube volume: $r = x \cdot \sqrt[3]{\frac{3}{4\pi}}$, where $x$ is the cube side length [S3, unofficial]. Vertices are counted in each neighborhood, and counts are normalized by dividing by the maximum count to scale between 0 and 1. These normalized values are attached as a field to the voxels [S3, unofficial]. A key pitfall is that using a spherical query for cubic voxels yields imperfect boundary results, as the sphere extends beyond the cube faces [S3, unofficial].

## Operating characteristics
- **Time Complexity**: Building the KDTree is $O(N \log N)$ [S1, S3, unofficial]. Querying neighbors for all points is $O(N \log N)$ on average for uniform distributions, but can degrade to $O(N^2)$ in worst-case scenarios (e.g., high dimensionality or clustered data) [S1, S3, unofficial].
- **Memory Complexity**: $O(N)$ to store the points and the KDTree structure [S1, S3, unofficial].
- **Strengths**: Simple to implement; provides a direct measure of local density; efficient for static point clouds [S1, S3, unofficial].
- **Limitations**: Sensitive to the choice of radius; spherical queries on cubic voxels introduce boundary artifacts [S3, unofficial]; does not account for point orientation or normals unless explicitly extended [S2, unofficial].
- **When to use**: When a quick estimate of local point density is needed for preprocessing, clustering, or density-based rendering techniques [S1, S3, unofficial].
- **When not to use**: When precise volumetric occupancy within cubic boundaries is required without boundary artifacts [S3, unofficial]; when point normals are critical for density estimation [S2, unofficial].

## Background
Neighbor Counting is a standard technique in point cloud processing and spatial data structures. It is commonly implemented using KDTree or similar spatial indexing structures [S1, S3, unofficial]. The method is widely used in applications such as point cloud denoising, clustering, and density-based rendering [S1, S3, unofficial]. The specific implementation details for voxel-based density evaluation using KDTree and spherical queries are documented in tutorials on mesh and point cloud voxelization [S3, unofficial]. The clusterability check variant is part of the KdtreeISO project for efficient mesh representation [S2, unofficial].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
[S3] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
