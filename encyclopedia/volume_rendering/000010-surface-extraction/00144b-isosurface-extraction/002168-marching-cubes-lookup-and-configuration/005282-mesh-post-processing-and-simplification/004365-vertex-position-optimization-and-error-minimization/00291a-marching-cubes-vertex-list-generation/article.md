# Marching Cubes Vertex List Generation

Also called: Edge Vertex Calculation, Marching Cubes Vertex Interpolation, Vertex Interpolation for Marching Cubes

## Overview
This method calculates the precise 3D positions of vertices lying on the edges of a voxel cube where the scalar field crosses a specified isolevel, using linear interpolation between corner values [S1, unofficial]. It generates the list of vertices for the polygon(s) inside a voxel based on the cube index, forming the geometric foundation for the subsequent triangle assembly steps defined in the parent algorithm [S1, unofficial]. Unlike sibling methods that optimize vertex positions via QEF or error minimization, this method relies strictly on linear interpolation along the grid edges [S1, unofficial].

## How it works
**Inputs:**
- `cubeIndex`: An integer (0–255) representing the configuration of the voxel, derived from the `edgeTable` lookup [Lookup Table and Ambiguity Handling](10327-lookup-table-and-ambiguity-handling.md).
- `voxelLocalPosition`: The 3D world position of the voxel's origin (corner 0).
- `voxelDensities`: An array of 8 scalar values (densities) corresponding to the 8 corners of the voxel, indexed 0 to 7.
- `isolevel`: The threshold density value where the surface is defined.

**Outputs:**
- `vertexList`: A list of 3D points (up to 12, though typically fewer per cube) representing the intersection points on the intersected edges.

**Data Structures:**
- `EdgeIndexTable`: An array of integers where entries at indices $2i$ and $2i+1$ give the indices of the two corners forming edge $i$ (for $i=0..11$) [S1, unofficial].
- `CubeCorners`: An array of 8 3D vectors representing the local coordinates of the 8 corners of a unit cube, indexed 0 to 7 [S1, unofficial].

**Steps:**
1. Initialize an empty `vertexList`.
2. Iterate $i$ from 0 to 11 (representing the 12 edges of a cube).
3. Check if the $i$-th bit of `cubeIndex` is set. If not, continue to the next edge.
4. If the bit is set, the edge is intersected:
   a. Retrieve the start and end corner indices from `EdgeIndexTable` at positions $2i$ and $2i+1$. Let these be $c_1$ and $c_2$.
   b. Calculate the world positions of these corners:
      $$ p_1 = \text{voxelLocalPosition} + \text{CubeCorners}[c_1] $$
      $$ p_2 = \text{voxelLocalPosition} + \text{CubeCorners}[c_2] $$
   c. Retrieve the densities for the two corners from `voxelDensities`: $v_1 = \text{voxelDensities}[c_1]$ and $v_2 = \text{voxelDensities}[c_2]$.
   d. Normalize densities if necessary (e.g., divide by 255 if stored as bytes) to match the `isolevel` scale [S1, unofficial].
   e. Calculate the intersection vertex $p$ using linear interpolation:
      $$ p = p_1 + \frac{(isolevel - v_1)}{(v_2 - v_1)} (p_2 - p_1) $$
   f. Add $p$ to `vertexList`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vertex_list(cube_index, voxel_local_position, voxel_densities, isolevel, edge_index_table, cube_corners):
    vertex_list = []
    
    # Iterate over the 12 edges of the cube
    for i in range(12):
        # Check if the i-th bit is set in cube_index
        if (cube_index >> i) & 1:
            # Get corner indices for this edge
            c1 = edge_index_table[2 * i]
            c2 = edge_index_table[2 * i + 1]
            
            # Get world positions of corners
            p1 = voxel_local_position + cube_corners[c1]
            p2 = voxel_local_position + cube_corners[c2]
            
            # Get densities
            v1 = voxel_densities[c1]
            v2 = voxel_densities[c2]
            
            # Handle potential division by zero if v1 == v2
            # [G] In practice, if v1 == v2, the edge does not cross the isolevel 
            # unless v1 == isolevel, which is a degenerate case. 
            # Standard MC assumes distinct values or handles equality via topology rules.
            if v2 == v1:
                # Fallback: midpoint if densities are equal (degenerate)
                # [G] This is a safeguard; standard MC relies on strict inequality for edge crossing.
                p = (p1 + p2) / 2.0
            else:
                # Linear interpolation
                t = (isolevel - v1) / (v2 - v1)
                p = p1 + t * (p2 - p1)
            
            vertex_list.append(p)
            
    return vertex_list
```

**Parameters:**
- `isolevel`: The density level where a surface is created. Densities below this are inside (solid), above are outside (air) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** Occurs if $v_1 = v_2$. The code does not explicitly handle this in the source, implying inputs are assumed distinct or the case is degenerate/unhandled [S1, unofficial].
- **Boundary Conditions:** The method assumes the voxel is part of a larger grid; boundary handling is typically managed by the parent algorithm's chunking or boundary cap logic [Marching Cubes Boundary Cap](13166-marching-cubes-boundary-cap.md).

## Subtopics

### [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md)
This method computes the 3D position of points on an isosurface (value 0) within a voxel grid by identifying grid edges that cross the isosurface and calculating their intersection points. It operates as a compute shader kernel that extracts surface points for a voxel grid by determining which edges of a cube are intersected and calculating the intersection point using one of three strategies: Quadratic Error Function (QEF), Centroid, or Blocky (cube center). It aggregates intersections into a single point per voxel, prioritizing computational efficiency on the GPU over topological correctness.

### Vertex-centered Data Interpolation
A technique to transition from cell-centered to vertex-centered data by 'diffusing' data values from cells to vertices using interpolation, preparing data for Marching Cubes [S2]. It derives a vertex's value from its adjacent cells using bi-linear or tri-linear interpolation, contingent on the data's dimensionality [S2]. For a vertex, the value is calculated using neighbor cell values (e.g., a vertex with value 6 is calculated using neighbors 8, 6, 6, and 4) [S2]. After interpolation, the size of each dimension of the vertex data increases by one relative to the original cell-centered data [S2]. A pitfall is that re-sampled vertex-centered data leads to conflicts between AMR levels, specifically dangling nodes, resulting in discontinuities or cracks in the iso-surfaces [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, as it involves a fixed number of operations (12 edge checks and interpolations) regardless of the voxel's content.
- **Memory Complexity:** $O(1)$ per voxel, storing up to 12 vertices.
- **Strengths:** Simple to implement, computationally inexpensive, and produces geometrically accurate linear approximations of the isosurface within each voxel.
- **Limitations:** Does not account for curvature within the voxel, leading to faceted surfaces. Sensitive to noise in the scalar field. Does not handle degenerate cases (e.g., $v_1 = v_2$) robustly without additional logic [S1, unofficial].
- **When to Use:** When computational efficiency is paramount and the scalar field is smooth. It serves as the baseline for more complex vertex optimization techniques.
- **When Not to Use:** When high geometric fidelity is required without post-processing, or when the scalar field contains significant noise or flat regions crossing the isolevel.

## Background
The method is a fundamental component of the Marching Cubes algorithm, originally described by Lorensen and Cline in 1987 [S2]. It relies on the topological configuration determined by the `edgeTable` and `triangleTable` [Lookup Table and Ambiguity Handling](10327-lookup-table-and-ambiguity-handling.md). The linear interpolation formula is a standard technique in computer graphics for finding intersections between a scalar field and a line segment. The specific implementation details provided here are derived from modern GPU-based voxel terrain generation systems [S1, unofficial].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] Analyzing Impact of Data Reduction Techniques on Visualization for AMR Applications Using AMReX Framework, https://doi.org/10.1145/3624062.3625123, fetched 2026-10-27
