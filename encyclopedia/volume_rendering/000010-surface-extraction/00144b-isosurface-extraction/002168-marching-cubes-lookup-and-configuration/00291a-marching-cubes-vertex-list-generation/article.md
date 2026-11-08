# Marching Cubes Vertex List Generation

Also called: Edge Vertex Calculation, Marching Cubes Vertex Interpolation, Vertex Interpolation for Marching Cubes

## Overview
Marching Cubes Vertex List Generation calculates the precise 3D positions of vertices lying on the edges of a voxel grid where a scalar field crosses a specified isovalue, producing a list of vertices for each cell [S1, unofficial]. It solves the geometric interpolation step required to convert discrete voxel data into a continuous polygonal surface, serving as the foundational vertex generation phase for the broader Marching Cubes algorithm [S1, unofficial]. Unlike [Isosurface extraction](30-isosurface-extraction.md) methods that may output point clouds or blocky approximations, this method specifically computes linearly interpolated intersection points to ensure topological continuity between adjacent cells.

## How it works
**Inputs and Outputs**
*   **Input:** A scalar field value (density) for each of the 8 corners of a voxel cell, the world position of the voxel's origin, and a scalar threshold (isolevel).
*   **Output:** A list of 3D vertex positions located on the edges of the voxel where the scalar field crosses the isolevel.

**Data Structures**
*   **EdgeIndexTable:** An array of integers where entries at indices $2i$ and $2i+1$ give the indices of the two corners forming edge $i$ (for $i=0..11$) [S1, unofficial].
*   **CubeCorners:** An array of 8 integer vectors (or coordinates) representing the local relative positions of the 8 corners of a unit cube, indexed 0 to 7 [S1, unofficial].

**Steps**
1.  Determine the **edge index** (or cube index) by comparing each of the 8 voxel corner densities against the isolevel. If a corner's density is below the isolevel, the corresponding bit is set; otherwise, it is unset. This results in an integer from 0 to 255.
2.  Initialize an empty list for vertices.
3.  Iterate through the 12 edges of the cube (index $i$ from 0 to 11).
4.  Check if the $i$-th bit of the edge index is set. If not, the surface does not intersect this edge; continue to the next edge.
5.  If the bit is set, the surface intersects this edge. Retrieve the start and end corner indices for edge $i$ from the `EdgeIndexTable` (at positions $2i$ and $2i+1$) [S1, unofficial].
6.  Calculate the world positions of these two corners ($p_1$ and $p_2$) by adding the voxel's local position to the corresponding entries in `CubeCorners` [S1, unofficial].
7.  Retrieve the scalar densities ($v_1$ and $v_2$) for these two corners. If the input data is normalized (e.g., 0-255), divide by the maximum value to normalize to [0,1] [S1, unofficial].
8.  Calculate the intersection vertex position using linear interpolation (see Formula).
9.  Add the calculated vertex to the vertex list.
10. Repeat for all 12 edges.

**Formula**
The position $p$ of the vertex on the edge is calculated as [S1, unofficial]:

$$ p = p_1 + \frac{(isolevel - v_1)}{(v_2 - v_1)} (p_2 - p_1) $$

Where:
*   $p_1, p_2$: The 3D world positions of the two endpoints of the edge.
*   $v_1, v_2$: The scalar density values at $p_1$ and $p_2$, respectively.
*   $isolevel$: The threshold density value defining the surface.

**Parameters**
*   **isolevel:** The density level where the surface is created. Densities below this are considered inside (solid), and densities above are outside (air) [S1, unofficial].

**Pitfalls and Edge Cases**
*   **Division by Zero:** If $v_1 = v_2$, the denominator in the interpolation formula becomes zero. The source implementation does not explicitly handle this, implying inputs are assumed to be distinct or the case is treated as degenerate/unhandled [S1, unofficial]. In practice, a small epsilon or a fallback to the midpoint should be used if exact equality is possible.
*   **Precision:** Linear interpolation assumes the scalar field varies linearly between vertices. This is an approximation; complex fields may result in slight geometric inaccuracies [G].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vertex_list(voxel_origin, voxel_densities, isolevel, edge_index_table, cube_corners):
    """
    Generates a list of vertex positions for a single voxel cell using Marching Cubes interpolation.
    
    Args:
        voxel_origin (Vector3): The world position of the voxel's minimum corner (0,0,0).
        voxel_densities (list of float): Densities at the 8 corners of the voxel, indexed 0-7.
        isolevel (float): The threshold density.
        edge_index_table (list of int): Table mapping edge index i to corner indices (2i, 2i+1).
        cube_corners (list of Vector3): Local coordinates of the 8 cube corners.
        
    Returns:
        list of Vector3: List of interpolated vertex positions on intersected edges.
    """
    vertices = []
    
    # Step 1: Determine edge index (cube index)
    # Note: The specific bit mapping (which corner corresponds to which bit) 
    # depends on the convention used for cube_corners indexing. 
    # Assuming standard Marching Cubes convention where bit k corresponds to corner k.
    edge_index = 0
    for k in range(8):
        if voxel_densities[k] < isolevel:
            edge_index |= (1 << k)
            
    # Step 2-9: Iterate edges and interpolate
    for i in range(12):
        # Check if edge i is intersected
        if (edge_index >> i) & 1:
            # Get corner indices for edge i
            corner_idx_1 = edge_index_table[2 * i]
            corner_idx_2 = edge_index_table[2 * i + 1]
            
            # Get world positions of corners
            p1 = voxel_origin + cube_corners[corner_idx_1]
            p2 = voxel_origin + cube_corners[corner_idx_2]
            
            # Get densities
            v1 = voxel_densities[corner_idx_1]
            v2 = voxel_densities[corner_idx_2]
            
            # Handle division by zero
            if v2 == v1:
                # Fallback to midpoint if densities are equal
                vertex = (p1 + p2) / 2.0
            else:
                # Linear interpolation
                t = (isolevel - v1) / (v2 - v1)
                vertex = p1 + t * (p2 - p1)
                
            vertices.append(vertex)
            
    return vertices
```

## Subtopics

### [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md)
Extracts a single 3D position per voxel using QEF, Centroid, or Blocky strategies, prioritizing GPU efficiency over topological correctness.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel cell, as it involves a fixed number of operations (12 edge checks and interpolations) regardless of the total volume size. The total time is linear with respect to the number of voxels $N$.
*   **Memory Complexity:** $O(1)$ auxiliary space per voxel for storing the vertex list (at most 12 vertices).
*   **Strengths:** Simple to implement; provides geometrically accurate linear interpolation; forms the basis for watertight mesh generation when combined with proper triangle indexing.
*   **Limitations:** Does not handle degenerate cases (exact isovalue matches) robustly without additional logic; linear interpolation may not capture high-frequency details in the scalar field.
*   **When to use:** When generating a polygonal mesh from a voxel grid and topological correctness is required.
*   **When not to use:** When only a point cloud representation is needed (use [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md)) or when the scalar field is known to have many exact isovalue matches.

## Background
The method is a core component of the Marching Cubes algorithm, originally introduced by William E. Lorensen and Harvey E. Cline in 1987 [S1, unofficial]. It has become the standard technique for isosurface extraction in medical imaging and voxel-based terrain generation. The specific implementation details provided here are derived from modern C#/Unity-based implementations of the algorithm.

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
