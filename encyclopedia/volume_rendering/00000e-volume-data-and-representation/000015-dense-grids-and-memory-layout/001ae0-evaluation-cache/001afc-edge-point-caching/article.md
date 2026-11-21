# Edge Point Caching

Also called: GetEdgePoint, X/Y/ZEdgePoint

## Overview
Edge Point Caching eliminates redundant linear interpolation calculations for isosurface intersection points on cube edges shared between adjacent voxels in the Marching Cubes algorithm [S1, unofficial]. It differs from standard Marching Cubes by maintaining a lookup structure that ensures each geometric edge is interpolated only once, regardless of how many adjacent cubes share it [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field grid, an isovalue threshold, and a voxel processing loop iterating over $i, j, k$ indices [S1, unofficial].
**Outputs:** A list of triangles defining the isosurface [S1, unofficial].
**Data Structures:** The cache maps each unique grid edge to an integer index [S1, unofficial]. If an edge has not been computed, the cache value is initialized to -1 [S1, unofficial]. The exact memory layout (e.g., 3D array, hash map, or flat buffer) is Not given in the sources.
**Steps:**
1. Initialize the edge cache with -1 for all entries [S1, unofficial].
2. For each voxel $(i, j, k)$:
   a. Determine the cube case based on scalar values at the 8 corners [G].
   b. For each active edge in the case:
      i. Identify the edge uniquely by its global coordinates and corner indices [S1, unofficial].
      ii. Map the edge to a specific cache access function (`GetXEdgePoint`, `GetYEdgePoint`, or `GetZEdgePoint`) based on the corner indices [S1, unofficial].
      iii. Check if the cache contains a valid index (not -1) for this edge [S1, unofficial].
      iv. If valid, retrieve the cached point index.
      v. If invalid, compute the intersection point using linear interpolation between the two corner scalar values and the isovalue. Store the new global point index in the cache and return it [S1, unofficial].
   c. Construct triangles using the retrieved edge point indices [G].
**Formulas:** Linear interpolation for an edge between corners $A$ and $B$ with scalar values $S_A$ and $S_B$: $t = \frac{V - S_A}{S_B - S_A}$, where $V$ is the isovalue. The intersection point is $P = A + t(B - A)$ [G].
**Parameters:** `Isovalue`: The threshold scalar value for surface extraction [S1, unofficial]. Cache initialization value: -1 [S1, unofficial].
**Edge Cases:** Boundary edges require cache dimensions or coordinate mapping to account for grid boundaries to avoid out-of-bounds access [G]. Degenerate cases where scalar values equal the isovalue yield the corner point itself [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# EdgeCache stores a global point index for each edge.
# Initialized to -1. Dimensions cover grid boundaries.
EdgeCache = [[[[ -1 for _ in range(12) ] for _ in range(Z+1)] for _ in range(Y+1)] for _ in range(X+1)]
GlobalPoints = []

def GetXEdgePoint(x, y, z, edge_idx):
    if EdgeCache[x][y][z][edge_idx] != -1:
        return EdgeCache[x][y][z][edge_idx]
    val1 = ScalarField[x][y][z]
    val2 = ScalarField[x+1][y][z]
    t = (Isovalue - val1) / (val2 - val1) if val2 != val1 else 0.0
    point = (x + t, y, z)
    idx = len(GlobalPoints)
    GlobalPoints.append(point)
    EdgeCache[x][y][z][edge_idx] = idx
    return idx

def GetYEdgePoint(x, y, z, edge_idx):
    if EdgeCache[x][y][z][edge_idx] != -1:
        return EdgeCache[x][y][z][edge_idx]
    val1 = ScalarField[x][y][z]
    val2 = ScalarField[x][y+1][z]
    t = (Isovalue - val1) / (val2 - val1) if val2 != val1 else 0.0
    point = (x, y + t, z)
    idx = len(GlobalPoints)
    GlobalPoints.append(point)
    EdgeCache[x][y][z][edge_idx] = idx
    return idx

def GetZEdgePoint(x, y, z, edge_idx):
    if EdgeCache[x][y][z][edge_idx] != -1:
        return EdgeCache[x][y][z][edge_idx]
    val1 = ScalarField[x][y][z]
    val2 = ScalarField[x][y][z+1]
    t = (Isovalue - val1) / (val2 - val1) if val2 != val1 else 0.0
    point = (x, y, z + t)
    idx = len(GlobalPoints)
    GlobalPoints.append(point)
    EdgeCache[x][y][z][edge_idx] = idx
    return idx

def GetEdgePoint(x, y, z, edge_idx):
    # Maps to specific function based on edge orientation and corner indices
    if edge_idx < 3:
        return GetXEdgePoint(x, y, z, edge_idx)
    elif edge_idx < 6:
        return GetYEdgePoint(x, y, z, edge_idx - 3)
    else:
        return GetZEdgePoint(x, y, z, edge_idx - 6)

def ProcessVoxel(i, j, k):
    # Determine cube case based on scalar values at the 8 corners [G]
    case = GetCubeCase(i, j, k)
    # ActiveEdges[case] contains the local edge indices for this case [G]
    for edge_idx in ActiveEdges[case]:
        pt_idx = GetEdgePoint(i, j, k, edge_idx)
        # AddPointToTriangle(pt_idx) constructs triangles using the retrieved edge point indices [G]
```

## Subtopics

### [Edge Interpolation with Caching](2431-edge-interpolation-with-caching.md)
Extends Edge Point Caching by storing gradient normals alongside intersection points to ensure consistent shading across shared edges. It builds on [Edge Point Caching](891-edge-point-caching.md) by ensuring consistent shading across shared edges.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
Originated in implementations of the Marching Cubes algorithm to optimize performance [S1, unofficial]. The specific implementation details are found in the provided C++ source code [S1, unofficial].

## Sources
[S1] samsfisher/3d-model-xray, https://github.com/samsfisher/3d-model-xray, fetched 2026-10-27 (unofficial)
