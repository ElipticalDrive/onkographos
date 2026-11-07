# Marching Cubes Vertex List Generation

Also called: Edge Vertex Calculation, Marching Cubes Vertex Interpolation, Vertex Interpolation for Marching Cubes

## Overview
This method calculates the precise 3D coordinates of vertices lying on the edges of a voxel cube where the scalar field crosses a specified isolevel, using linear interpolation between corner values [S1, unofficial]. It generates the list of vertices for the polygon(s) inside a voxel based on the cube index, serving as the geometric foundation for subsequent triangle assembly [S1, unofficial]. Unlike its sibling [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md), which aggregates intersections into a single representative point per voxel for efficiency, this method preserves topological correctness by generating distinct vertices for each intersected edge, enabling proper polygonal connectivity [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `cubeIndex`: The 8-bit case index ($0 \le C \le 255$) derived from the voxel's corner signs relative to the isolevel.
    *   `voxelLocalPosition`: The 3D world position of the voxel's origin (corner 0).
    *   `voxelDensities`: An array of 8 scalar values ($v_0 \dots v_7$) representing the field values at the cube's corners.
    *   `isolevel`: The threshold density value $I$ at which the surface is defined.
    *   `edgeTable`: The 256-entry bitmask array from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) indicating which edges are intersected.
    *   `cubeCorners`: An array of 8 vectors representing the local coordinates of the unit cube corners (typically $(0,0,0)$ to $(1,1,1)$).
*   **Outputs**:
    *   `vertexList`: A list of 3D positions corresponding to the intersection points on the intersected edges. The size is equal to the number of set bits in `edgeTable[cubeIndex]`.

**Data Structures**
*   **Edge Index Table**: While the parent article defines `edgeTable` as a bitmask, the implementation requires mapping edge indices to corner pairs. Let `edgeCorners[i]` be a pair of integers $(a, b)$ for edge $i$ ($0 \le i \le 11$), where $a$ and $b$ are the indices of the two vertices forming edge $i$. This is implicitly defined by the standard edge numbering convention:
    *   $e_0=(0,1), e_1=(1,2), e_2=(2,3), e_3=(3,0)$
    *   $e_4=(4,5), e_5=(5,6), e_6=(6,7), e_7=(7,4)$
    *   $e_8=(0,4), e_9=(1,5), e_{10}=(2,6), e_{11}=(3,7)$
*   **Cube Corners**: `cubeCorners` is an array of 8 `float3` vectors. In a unit cube aligned with axes, `cubeCorners[k]` corresponds to the binary representation of $k$.

**Algorithm Steps**
1.  Initialize an empty list `vertexList`.
2.  Retrieve the intersection bitmask `mask = edgeTable[cubeIndex]`.
3.  Iterate through each edge index $i$ from 0 to 11:
    a.  Check if the $i$-th bit of `mask` is set (i.e., `(mask >> i) & 1`).
    b.  If set, calculate the intersection vertex for edge $i$ and append it to `vertexList`.
4.  Return `vertexList`.

**Vertex Interpolation Formula**
For an edge connecting corner 1 (position $p_1$, density $v_1$) and corner 2 (position $p_2$, density $v_2$), the intersection point $p$ is calculated as [S1, unofficial]:

$$ p = p_1 + \frac{(I - v_1)}{(v_2 - v_1)} (p_2 - p_1) $$

Where:
*   $I$ is the `isolevel`.
*   $p_1, p_2$ are the 3D world positions of the edge endpoints.
*   $v_1, v_2$ are the scalar field values at the edge endpoints.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vertex_list(cube_index, voxel_local_pos, voxel_densities, isolevel, edge_table, cube_corners):
    """
    Generates a list of vertex positions for the isosurface intersections within a single voxel.
    
    Args:
        cube_index (int): The 8-bit case index (0-255).
        voxel_local_pos (float3): The world position of the voxel's corner 0.
        voxel_densities (list[float]): Densities at corners 0-7.
        isolevel (float): The threshold value.
        edge_table (list[int]): 256-entry bitmask array.
        cube_corners (list[float3]): Local coordinates of the 8 unit cube corners.
        
    Returns:
        list[float3]: List of intersection vertex positions.
    """
    # Standard edge-to-corner mapping for Marching Cubes
    # Edge i connects corner edge_corners[i][0] and edge_corners[i][1]
    edge_corners = [
        (0, 1), (1, 2), (2, 3), (3, 0),  # Bottom face
        (4, 5), (5, 6), (6, 7), (7, 4),  # Top face
        (0, 4), (1, 5), (2, 6), (3, 7)   # Vertical edges
    ]
    
    vertex_list = []
    mask = edge_table[cube_index]
    
    for i in range(12):
        # Check if the i-th bit is set
        if (mask >> i) & 1:
            # Get corner indices for this edge
            c1_idx, c2_idx = edge_corners[i]
            
            # Get local corner positions
            local_p1 = cube_corners[c1_idx]
            local_p2 = cube_corners[c2_idx]
            
            # Convert to world positions
            p1 = voxel_local_pos + local_p1
            p2 = voxel_local_pos + local_p2
            
            # Get densities
            v1 = voxel_densities[c1_idx]
            v2 = voxel_densities[c2_idx]
            
            # Calculate intersection using linear interpolation
            # Handle division by zero if v1 == v2 [G]
            if v1 == v2:
                # Degenerate case: edge is parallel to isosurface or value is exactly isolevel.
                # Standard practice is to use the midpoint or one of the endpoints.
                # Using midpoint for stability.
                vertex = (p1 + p2) / 2.0
            else:
                t = (isolevel - v1) / (v2 - v1)
                vertex = p1 + t * (p2 - p1)
            
            vertex_list.append(vertex)
            
    return vertex_list
```

**Parameters**
*   `isolevel`: The density level where a surface is created. Densities below this are typically considered inside (solid), and above are outside (air), though this convention depends on the specific implementation [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero**: If $v_1 = v_2$, the denominator in the interpolation formula becomes zero. This occurs if the scalar field is constant along an edge or if both endpoints lie exactly on the isolevel. The code must handle this, typically by returning the midpoint or one of the endpoints [S1, unofficial].
*   **Precision**: Floating-point precision errors can cause vertices on shared edges between adjacent voxels to differ slightly, leading to visible cracks in the mesh. This is often addressed in later stages by snapping or merging vertices [G].
*   **Ambiguity**: The vertex positions themselves are not affected by the topological ambiguity of cases 46, 47, 98, 99, 210, 213, 214, 215. However, the subsequent triangle connection (handled by the parent's `triangleTable`) will determine how these vertices are connected, which may result in incorrect topology if the ambiguity is not resolved consistently across adjacent voxels [S1, unofficial].

## Subtopics

### [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md)
Extracts a single 3D position per voxel using QEF, Centroid, or Blocky strategies. Choose this when computational efficiency on the GPU is prioritized over topological correctness and point-based rendering is sufficient.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel, as it involves a fixed number of iterations (12 edges) and simple arithmetic operations.
*   **Memory Complexity**: $O(1)$ per voxel, storing up to 12 vertices.
*   **Strengths**: Simple to implement, mathematically exact for linear interpolation, preserves topological structure for mesh generation.
*   **Limitations**: Does not handle topological ambiguities; requires post-processing for vertex merging to avoid cracks; sensitive to floating-point precision.
*   **When to use**: As the core geometric step in standard Marching Cubes mesh generation pipelines.
*   **When not to use**: When point-based rendering is sufficient and mesh connectivity is not required (use [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md) instead).

## Background
The method is a fundamental component of the Marching Cubes algorithm, originally described by Lorensen and Cline in 1987. It relies on the linear interpolation assumption that the isosurface intersects cube edges at a point proportional to the scalar field values at the endpoints. The specific implementation details provided here are derived from modern practical implementations in voxel terrain generation [S1, unofficial].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
