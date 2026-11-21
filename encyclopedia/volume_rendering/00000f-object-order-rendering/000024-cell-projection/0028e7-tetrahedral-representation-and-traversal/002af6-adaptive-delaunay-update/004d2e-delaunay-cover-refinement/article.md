# Delaunay Cover Refinement
## Overview
Delaunay Cover Refinement is a mesh generation technique that iteratively subdivides a Delaunay triangulation to ensure all edge lengths are bounded by a user-defined threshold $h$ [S1, unofficial]. It solves the problem of generating a uniformly sampled point cloud from an initial point set, which serves as a prerequisite for subsequent voxelization or surface reconstruction steps [S1, unofficial]. Unlike standard Delaunay Triangulation which only optimizes local geometry, this method enforces a global density constraint through recursive splitting [S1, unofficial].
## How it works
**Inputs:**
- Initial point set $P$.
- Maximum edge length threshold $h$.

**Outputs:**
- Refined point cloud $P_\kappa$.
- Final Delaunay cover $DC_\kappa(P)$.

**Data Structures:**
- Point cloud $P_i$: A list of 3D coordinates.
- Delaunay cover $DC_i(P)$: A set of triangles (simplices) defined by vertices from $P_i$.

**Algorithm:**
1. Initialize $P_0 = P$ and compute the initial Delaunay cover $DC_0(P)$.
2. Iterate $i$ from 0 upwards:
   a. Identify all triangles in $DC_i(P)$ that contain at least one edge with length greater than $h$.
   b. For each such triangle, perform a 1-to-4 split:
      i. Calculate the midpoints of the three edges of the triangle.
      ii. Insert these three midpoints into the point set to form $P_{i+1}$.
      iii. Recompute the Delaunay triangulation for the updated point set to form $DC_{i+1}(P)$.
   c. If no triangles in $DC_i(P)$ have edges longer than $h$, terminate.
   d. Otherwise, increment $i$ and repeat.

**Formulas:**
- Edge length check: $\|v_j - v_k\| > h$ for vertices $v_j, v_k$ of a triangle.
- Midpoint calculation: $m = \frac{v_j + v_k}{2}$.

**Parameters:**
- $h$: The maximum allowed edge length. Typical values depend on the desired sampling density relative to the scene scale.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in midpoint calculation may cause slight deviations from exact geometric centers, potentially affecting Delaunay properties.
- **Boundaries:** The method assumes a bounded domain. The resulting point cloud is confined to the convex hull of the initial points.
- **Termination:** The process is guaranteed to terminate because each split reduces the maximum edge length in the affected region, and the number of points is finite for a bounded domain with fixed $h$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delaunay_cover_refinement(points, h):
    """
    Iteratively refine a Delaunay cover until all edges are <= h.
    
    Args:
        points: List of 3D points (initial P_0).
        h: Maximum edge length threshold.
        
    Returns:
        Refined point cloud P_kappa.
    """
    current_points = list(points)
    
    while True:
        # Compute Delaunay triangulation for current points
        # Note: In practice, use a robust library or incremental update.
        # Here we assume a function that returns a list of triangles.
        triangles = compute_delaunay_triangulation(current_points)
        
        # Check if any triangle has an edge longer than h
        needs_refinement = False
        new_midpoints = []
        
        for tri in triangles:
            v0, v1, v2 = tri
            edges = [(v0, v1), (v1, v2), (v2, v0)]
            
            for p1, p2 in edges:
                dist = vector_distance(p1, p2)
                if dist > h:
                    needs_refinement = True
                    # Calculate midpoint
                    mid = (p1 + p2) / 2.0
                    # Avoid duplicate midpoints if multiple edges share the same midpoint
                    # In a 1-to-4 split, all three midpoints are added simultaneously
                    # We collect unique midpoints for this iteration
                    if mid not in new_midpoints:
                        new_midpoints.append(mid)
        
        if not needs_refinement:
            break
            
        # Add new midpoints to the point set
        current_points.extend(new_midpoints)
        
    return current_points

def vector_distance(p1, p2):
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5
```
## Subtopics

### [Probing for Surface Mesh Generation through Delaunay Refinement](17029-probing-for-surface-mesh-generation-through-delaunay-refinem.md)
Generates surface meshes from implicit surfaces without seed points by combining probing with Delaunay refinement. It guarantees topological equivalence and captures isolated components autonomously. Choose this when initial point sets are unavailable or when capturing disconnected surface components is required.

## Operating characteristics
- **Time Complexity:** Not explicitly given, but depends on the number of refinement iterations $\kappa$ and the cost of recomputing the Delaunay triangulation at each step.
- **Memory Complexity:** Proportional to the number of points in the final cloud $P_\kappa$.
- **Strengths:** Guarantees uniform sampling density bounded by $h$. Simple to implement.
- **Limitations:** Requires recomputing the Delaunay triangulation iteratively, which can be computationally expensive for large datasets. Does not inherently handle non-convex geometries without additional constraints.
- **When to use:** When a uniformly sampled point cloud is needed for voxelization or as input for surface reconstruction algorithms that require bounded edge lengths.
## Background
Originated in the context of volumetric snapping and watertight triangulation of point clouds [S1, unofficial]. The method is part of a pipeline that converts point clouds into voxel grids for volume rendering applications [S1, unofficial].
## Sources
[S1] Volumetric Snapping: Watertight Triangulation of Point Clouds, https://www.scitepress.org/PublishedPapers/2007/20798/pdf/index.html, fetched 2026-10-27 (unofficial)
