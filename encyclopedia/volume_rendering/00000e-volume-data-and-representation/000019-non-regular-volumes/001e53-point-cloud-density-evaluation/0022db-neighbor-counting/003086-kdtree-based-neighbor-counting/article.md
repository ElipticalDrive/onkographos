# KDTree-based Neighbor Counting

Also called: KDTree Density Evaluation, Per-Voxel Point Density Calculation via KDTree

## Overview
KDTree-based Neighbor Counting calculates the density of mesh vertices within each voxel by querying neighboring points using a KDTree and normalizing the counts [S1, unofficial]. Comparisons to sibling methods are not given in the sources.

## How it works
**Inputs:** A set of mesh vertices or point cloud samples $P$, and a voxel grid with side length $x$ [S1, unofficial].
**Outputs:** A voxel grid where each voxel stores a normalized density value $d \in [0, 1]$ [S1, unofficial].

**Data Structures:**
1. **KDTree:** A spatial index built from $P$, implemented via SciPy's `cKDTree` [G].
2. **Voxel Grid:** A collection of voxel centers $C = \{c_1, \dots, c_M\}$, each with a `density` field [G].

**Steps:**
1. **Calculate Query Radius:** Compute the radius $r$ of a sphere with the same volume as the voxel cube: $r = x \left( \frac{3}{4\pi} \right)^{1/3}$ [S1, unofficial].
2. **Build KDTree:** Construct the spatial index from $P$ [G].
3. **Query Neighbors:** For each voxel center $c_i$, find all points within distance $r$ using `query_ball_point` [S1, unofficial].
4. **Count Vertices:** Count the number of points in each neighborhood [S1, unofficial].
5. **Normalize:** Divide each count by the maximum count across all voxels to scale values between 0 and 1 [S1, unofficial].
6. **Attach Field:** Assign the normalized values as a field to the voxels [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import cKDTree

def kdtree_neighbor_counting(points, voxel_centers, voxel_side_length):
    """
    Calculate normalized point density for each voxel using KDTree.
    
    Args:
        points: Array of shape (N, 3) containing mesh vertices or point cloud samples.
        voxel_centers: Array of shape (M, 3) containing the center coordinates of each voxel.
        voxel_side_length: Float, the side length x of the cubic voxels.
        
    Returns:
        densities: Array of shape (M,) containing normalized density values in [0, 1].
    """
    # Step 1: Calculate the equivalent-volume sphere radius
    r = voxel_side_length * (3.0 / (4.0 * np.pi)) ** (1.0 / 3.0)
    
    # Step 2: Build KDTree from input points
    tree = cKDTree(points)
    
    # Step 3 & 4: Query neighbors and count
    neighborhoods = tree.query_ball_point(voxel_centers, r)
    counts = np.array([len(nbhd) for nbhd in neighborhoods])
    
    # Step 5: Normalize
    max_count = np.max(counts)
    if max_count == 0:
        return np.zeros_like(counts, dtype=float)
    
    densities = counts / max_count
    
    # Step 6: Return densities (in a full implementation, these would be attached to voxel objects)
    return densities
```

**Parameters:**
* `voxel_side_length` ($x$): Determines grid resolution and query radius. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
* **Boundary Imperfection:** Using a spherical query for cubic voxels yields imperfect boundary results, as the sphere extends beyond the cube faces [S1, unofficial].

## Subtopics

### [Efficient kNN Implementation](10966-efficient-knn-implementation.md)
Accelerates k-nearest neighbor searches in point cloud and Gaussian splatting pipelines by providing global and local search strategies. It sets itself apart from [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md) by leveraging camera parameters to restrict search spaces rather than using a fixed spherical radius. Choose it when scaling to large scenes or requiring camera-aware locality.

### [Kernel Density Estimation (KDE)](13917-kernel-density-estimation-kde.md)
Transforms discrete point clouds into smooth, continuous scalar fields to reveal interior structures obscured by noise or occlusion. It sets itself apart from [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md) by providing density smoothing and gradient-driven lighting rather than discrete neighbor counts. Choose it when continuous volumetric representation is required over exact indexing.

## Operating characteristics
Not given in the sources.

## Background
Described in a tutorial on voxelizing meshes and point clouds in Python using PyVista and SciPy [S1, unofficial]. It is presented as a useful analysis step for creating interactive GUIs for mesh thresholding, leveraging SciPy's `query_ball_point` functionality.

## Sources
[S1] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
