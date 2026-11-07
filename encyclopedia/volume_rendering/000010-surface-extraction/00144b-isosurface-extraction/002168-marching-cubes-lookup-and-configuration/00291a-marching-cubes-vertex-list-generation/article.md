# Marching Cubes Vertex List Generation

Also called: Edge Vertex Calculation, Marching Cubes Vertex Interpolation, Vertex Interpolation for Marching Cubes

## Overview
This method calculates the precise 3D coordinates of vertices where the isosurface intersects the edges of a voxel cube, using linear interpolation between the scalar field values at the cube's corners [S1, unofficial]. It serves as the geometric core of the Marching Cubes algorithm, transforming the topological information provided by the parent [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) into actual spatial data. Unlike [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md), which aggregates intersections into single representative points for point-based rendering, this method generates distinct vertices for each intersected edge to preserve polygonal connectivity for mesh generation.

## How it works
**Inputs and Outputs**
*   **Input:**
    *   `cubeIndex`: The 8-bit case index ($0 \le cubeIndex \le 255$) derived from the voxel's corner signs relative to the isolevel.
    *   `cornerPositions`: An array of 8 3D vectors representing the world coordinates of the voxel's corners, indexed $0$ to $7$.
    *   `cornerValues`: An array of 8 scalar values (densities) corresponding to the corners, indexed $0$ to $7$.
    *   `isolevel`: The threshold scalar value defining the surface.
    *   `edgeTable`: The 256-entry bitmask array from the parent article, indicating which edges are intersected.
*   **Output:**
    *   `vertexList`: A list of 3D vectors containing the interpolated intersection points. The size is variable (0 to 12 vertices), but typically small.

**Data Structures**
*   **Edge Index Table (`edgeIndexTable`)**: An array of 24 integers (12 pairs). For each edge $i$ ($0 \le i \le 11$), the pair at indices $2i$ and $2i+1$ contains the indices of the two corner vertices that form edge $i$. This maps the abstract edge index to the concrete corner indices.
    *   *Note:* The standard Marching Cubes convention defines the 12 edges. The `edgeIndexTable` must be consistent with the vertex ordering used to generate `cubeIndex`.
*   **Cube Corners (`cubeCorners`)**: An array of 8 3D vectors representing the local coordinates of a unit cube (e.g., $(0,0,0)$ to $(1,1,1)$). These are used to convert local edge definitions to world space if `cornerPositions` are not pre-computed in world space. However, the implementation in [S1, unofficial] assumes `cornerPositions` are already in world space.

**Algorithm Steps**
1.  Initialize an empty list `vertexList`.
2.  Iterate through each edge index $i$ from $0$ to $11$.
3.  Check if the $i$-th bit of `edgeTable[cubeIndex]` is set.
    *   If the bit is **0**, the isosurface does not intersect this edge. Continue to the next edge.
    *   If the bit is **1**, the isosurface intersects this edge. Proceed to step 4.
4.  Retrieve the corner indices for edge $i$ from `edgeIndexTable`:
    *   `startCornerIndex = edgeIndexTable[2 * i]`
    *   `endCornerIndex = edgeIndexTable[2 * i + 1]`
5.  Retrieve the scalar values for these corners:
    *   $v_1 = \text{cornerValues}[\text{startCornerIndex}]$
    *   $v_2 = \text{cornerValues}[\text{endCornerIndex}]$
6.  Retrieve the world positions for these corners:
    *   $p_1 = \text{cornerPositions}[\text{startCornerIndex}]$
    *   $p_2 = \text{cornerPositions}[\text{endCornerIndex}]$
7.  Calculate the intersection position $p$ using linear interpolation:
    $$ p = p_1 + \frac{(isolevel - v_1)}{(v_2 - v_1)} (p_2 - p_1) $$
8.  Append $p$ to `vertexList`.
9.  After iterating through all 12 edges, return `vertexList`.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vertex_list(cube_index, corner_positions, corner_values, isolevel, edge_table, edge_index_table):
    """
    Generates a list of 3D vertices where the isosurface intersects the voxel edges.
    
    Args:
        cube_index (int): 8-bit case index (0-255).
        corner_positions (list of vec3): 8 world positions of the cube corners.
        corner_values (list of float): 8 scalar values at the cube corners.
        isolevel (float): The threshold value for the isosurface.
        edge_table (list of int): 256-entry bitmask array.
        edge_index_table (list of int): 24-entry array mapping edge index to corner indices.
        
    Returns:
        list of vec3: List of interpolated vertex positions.
    """
    vertex_list = []
    
    # Get the bitmask for the current cube configuration
    edge_mask = edge_table[cube_index]
    
    for i in range(12):
        # Check if the i-th bit is set in the edge mask
        if (edge_mask >> i) & 1:
            # Get the two corner indices for this edge
            start_idx = edge_index_table[2 * i]
            end_idx = edge_index_table[2 * i + 1]
            
            # Get corner values and positions
            v1 = corner_values[start_idx]
            v2 = corner_values[end_idx]
            p1 = corner_positions[start_idx]
            p2 = corner_positions[end_idx]
            
            # Avoid division by zero if values are equal
            # [G] If v1 == v2, the edge is parallel to the isosurface or the value is exactly the isolevel.
            # Standard MC assumes distinct values for intersection. If v1 == v2, the intersection is undefined
            # by linear interpolation. A common fallback is to use the midpoint, or skip if strictly equal.
            # However, the source [S1, unofficial] does not explicitly handle this, implying inputs are distinct.
            if v1 == v2:
                # Degenerate case: edge is parallel to isosurface.
                # Use midpoint as a safe fallback for implementation robustness.
                p = (p1 + p2) / 2.0
            else:
                # Linear interpolation formula
                t = (isolevel - v1) / (v2 - v1)
                p = p1 + t * (p2 - p1)
            
            vertex_list.append(p)
            
    return vertex_list
```

**Parameters**
*   `isolevel`: The scalar threshold. In [S1, unofficial], voxel data is normalized to $[0, 1]$ by dividing by 255, so the isolevel is also in $[0, 1]$. Densities below the isolevel are considered "inside" (solid), and above are "outside" (air) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero**: If $v_1 = v_2$, the denominator in the interpolation formula is zero. This occurs if both corners have the same scalar value. The source [S1, unofficial] does not explicitly handle this, implying that inputs are assumed to be distinct or such cases are degenerate. In practice, if $v_1 = v_2 = isolevel$, the entire edge lies on the surface; if $v_1 = v_2 \neq isolevel$, the edge does not intersect. A robust implementation should handle this, e.g., by using the midpoint or skipping [G].
*   **Precision**: Floating-point precision errors can cause slight mismatches in shared vertices between adjacent voxels if the interpolation is not performed identically. Using the same `edgeIndexTable` and consistent corner ordering is critical.
*   **Ambiguity**: The vertex positions themselves are not affected by the topological ambiguity (cases 46, 47, etc.), but the connectivity (triangles) is. The vertex list generation is purely geometric and does not resolve ambiguity.

## Subtopics

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel. The algorithm iterates through a fixed number of 12 edges, performing constant-time arithmetic operations.
*   **Memory Complexity**: $O(1)$ per voxel. The output list size is bounded by 12 vertices.
*   **Performance**: Extremely fast, suitable for real-time applications. The primary cost is memory access to `cornerValues` and `cornerPositions`.
*   **Strengths**: Simple, deterministic, and geometrically accurate for linear scalar fields.
*   **Limitations**: Does not handle topological ambiguity. Requires consistent corner ordering and edge indexing across adjacent voxels to avoid cracks.

## Background
The method is a fundamental component of the Marching Cubes algorithm, originally described by Lorensen and Cline in 1987. The specific implementation details, including the use of `edgeIndexTable` and the linear interpolation formula, are widely adopted in computer graphics for isosurface extraction. The source [S1, unofficial] provides a practical implementation in the context of voxel terrain generation.

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
