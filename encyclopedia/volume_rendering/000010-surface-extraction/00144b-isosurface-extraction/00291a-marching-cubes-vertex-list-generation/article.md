# Marching Cubes Vertex List Generation

Also called: Edge Vertex Calculation, Marching Cubes Vertex Interpolation, Vertex Interpolation for Marching Cubes

## Overview
This method computes the precise 3D positions of vertices on an isosurface within a voxel grid by identifying grid edges that cross the isosurface and calculating their intersection points via linear interpolation [S1, unofficial]. It serves as the geometric refinement step following [Marching Cubes Vertex List Generation](4447-marching-cubes-vertex-list-generation.md), which determines which edges are intersected but does not compute the exact surface location [S1, unofficial]. Unlike [Surface Nets](6419-surface-nets.md), which approximates the surface point using the grid vertex with the highest gradient, this method offers three distinct strategies for point placement: Quadratic Error Function (QEF) minimization, simple centroid averaging, or a blocky (cube center) fallback [S1, unofficial].

## How it works
**Inputs:**
*   `voxelLocalPosition`: The 3D world coordinate of the current voxel's origin (typically the minimum corner).
*   `voxelDensities`: An array of 8 scalar values representing the density at each corner of the voxel, indexed 0 to 7.
*   `edgeIndex`: A 12-bit integer (cube index) indicating which of the 12 edges of the voxel are intersected by the isosurface.
*   `isolevel`: The scalar threshold value defining the surface.

**Outputs:**
*   `vertexList`: A list of 3D positions (up to 12 vertices) corresponding to the intersection points on the intersected edges.

**Data Structures:**
*   `EdgeIndexTable`: An array of integers where entries at indices $2i$ and $2i+1$ give the indices of the two corners forming edge $i$ (for $i=0..11$) [S1, unofficial].
*   `CubeCorners`: An array of 8 3D vectors representing the local coordinates of the 8 corners of a unit cube, indexed 0 to 7 [S1, unofficial].

**Steps:**
1.  Initialize an empty list `vertexList`.
2.  Iterate $i$ from 0 to 11 (representing the 12 edges of a cube).
3.  If the $i$-th bit of `edgeIndex` is set:
    a.  Retrieve the start and end corner indices from `EdgeIndexTable` at positions $2i$ and $2i+1$.
    b.  Calculate the world positions of these corners by adding `voxelLocalPosition` to `CubeCorners[startIndex]` and `CubeCorners[endIndex]`.
    c.  Retrieve the densities for the two corners of the edge from `voxelDensities`.
    d.  Normalize the densities and isolevel to the range $[0, 1]$ if they are stored as integers (e.g., dividing by 255) [S1, unofficial].
    e.  Call `VertexInterpolate` with the two corner positions, their normalized densities, and the normalized isolevel.
    f.  Store the result in `vertexList`.

**Formula:**
The position $p$ of the vertex on the edge is calculated using linear interpolation:

$$
p = p_1 + \frac{(isolevel - v_1)}{(v_2 - v_1)} (p_2 - p_1)
$$

Where:
*   $p_1, p_2$ are the 3D positions of the two corners of the edge.
*   $v_1, v_2$ are the scalar density values at the two corners.
*   $isolevel$ is the threshold density.

**Parameters:**
*   `isolevel`: The density level where a surface is created. Densities below this are inside (solid), above are outside (air) [S1, unofficial].

**Pitfalls:**
*   Division by zero occurs if $v_1 = v_2$. The code does not explicitly handle this, implying inputs are assumed distinct or the case is degenerate/unhandled [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vertex_list(voxel_local_position, voxel_densities, edge_index, isolevel, edge_index_table, cube_corners):
    """
    Generates the list of vertices for the polygon(s) inside a voxel based on the cube index.
    
    Args:
        voxel_local_position: vec3, world position of the voxel's origin.
        voxel_densities: list of 8 floats, density at each corner of the voxel.
        edge_index: int, 12-bit mask indicating intersected edges.
        isolevel: float, the threshold density.
        edge_index_table: list of 24 ints, mapping edge index to corner indices.
        cube_corners: list of 8 vec3, local coordinates of cube corners.
        
    Returns:
        list of vec3, the interpolated vertex positions on intersected edges.
    """
    vertex_list = []
    
    # Normalize values if necessary (assuming 0-255 integer input)
    # Note: This normalization step is specific to the source implementation [S1, unofficial]
    normalized_isolevel = isolevel / 255.0
    
    for i in range(12):
        # Check if the i-th edge is intersected
        if (edge_index >> i) & 1:
            # Get corner indices for this edge
            start_idx = edge_index_table[2 * i]
            end_idx = edge_index_table[2 * i + 1]
            
            # Calculate world positions of the corners
            p1 = voxel_local_position + cube_corners[start_idx]
            p2 = voxel_local_position + cube_corners[end_idx]
            
            # Get densities and normalize
            v1 = voxel_densities[start_idx] / 255.0
            v2 = voxel_densities[end_idx] / 255.0
            
            # Interpolate vertex position
            vertex = vertex_interpolate(p1, p2, v1, v2, normalized_isolevel)
            vertex_list.append(vertex)
            
    return vertex_list

def vertex_interpolate(p1, p2, v1, v2, isolevel):
    """
    Calculates the intersection point on the edge between p1 and p2.
    
    Formula: p = p1 + ((isolevel - v1) / (v2 - v1)) * (p2 - p1)
    """
    # Handle degenerate case where v1 == v2 to avoid division by zero [S1, unofficial]
    if v1 == v2:
        # Fallback to midpoint or handle as error. 
        # Source implies this case is unhandled/degenerate [S1, unofficial].
        # A safe implementation might return the midpoint.
        return (p1 + p2) / 2.0
        
    t = (isolevel - v1) / (v2 - v1)
    return p1 + t * (p2 - p1)
```

## Subtopics

### [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md)
This method computes the precise 3D position of points on an isosurface (typically value 0) within a voxel grid by identifying grid edges that cross the isosurface and calculating their intersection points. It serves as the geometric refinement step following [Marching Cubes Vertex List Generation](4447-marching-cubes-vertex-list-generation.md), which determines which edges are intersected but does not compute the exact surface location. Unlike [Surface Nets](6419-surface-nets.md), which approximates the surface point using the grid vertex with the highest gradient, this method offers three distinct strategies for point placement: Quadratic Error Function (QEF) minimization, simple centroid averaging, or a blocky (cube center) fallback.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel, as it involves a fixed number of operations (12 edge checks and interpolations) regardless of the total volume size.
*   **Memory Complexity:** $O(1)$ auxiliary memory per voxel for storing the vertex list (up to 12 vertices).
*   **Strengths:** Provides geometrically accurate surface positions by leveraging linear interpolation, resulting in smoother surfaces compared to blocky methods.
*   **Limitations:** Sensitive to noise in the scalar field, which can lead to jagged or noisy surfaces. Requires careful handling of degenerate cases (e.g., $v_1 = v_2$) to avoid numerical instability [S1, unofficial].
*   **When to use:** When high-quality, smooth isosurfaces are required from a voxel grid and computational cost per voxel is acceptable.
*   **When not to use:** When real-time performance on extremely large volumes is critical and simpler, faster approximations (like Surface Nets) are sufficient, or when the scalar field contains significant noise that would require pre-filtering.

## Background
The method is based on the classic Marching Cubes algorithm introduced by Lorensen and Cline in 1987 [S1, unofficial]. It is a fundamental technique in volume rendering and scientific visualization for extracting polygonal meshes from scalar data. The specific implementation details described here are derived from the Eldemarkki Marching Cubes Terrain project, which adapts the classic algorithm for real-time voxel terrain generation [S1, unofficial].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
