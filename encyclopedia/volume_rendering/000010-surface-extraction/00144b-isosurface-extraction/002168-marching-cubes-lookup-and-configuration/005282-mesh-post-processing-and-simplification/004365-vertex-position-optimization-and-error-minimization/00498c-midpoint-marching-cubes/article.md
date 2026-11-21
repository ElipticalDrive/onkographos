# Midpoint Marching Cubes

Also called: Proposed Marching Cubes Algorithm

## Overview
Midpoint Marching Cubes is a surface extraction method that modifies the standard Marching Cubes algorithm by replacing linear scalar interpolation on cube edges with fixed midpoint selection [S1, unofficial]. It reduces the 256 possible cube configurations to 15 basic patterns using reflective and rotational symmetries, thereby eliminating the need for scalar value interpolation during the intersection calculation phase [S1, unofficial]. Unlike its siblings which optimize vertex positions or handle complex ambiguities, this method prioritizes computational simplicity by fixing intersection points to geometric centers, though it sacrifices accuracy for complex geometries and suffers from the "hole problem" due to reliance on complementary symmetry alone [S1, unofficial].

## How it works
**Inputs:** A regular 3-D grid derived from a volumetric dataset (e.g., endoscope scanning data), where each grid vertex has an associated binary occupancy state (1 if inside the surface, 0 if outside) [S1, unofficial].
**Outputs:** A triangle mesh representing the isosurface, rendered via OpenGL [S1, unofficial].

**Data Structures:**
*   **Grid:** A 3D array of vertices. Each vertex $v$ at coordinates $(i, j, k)$ holds a binary value $b \in \{0, 1\}$ [S1, unofficial].
*   **Cube Index:** An 8-bit integer representing the occupancy state of the 8 vertices of a logical cube. The bits correspond to vertices #1 through #8 as defined in the standard MC configuration [S1, unofficial].
*   **Lookup Table:** A reduced table mapping the 8-bit index to one of 15 basic surface intersection patterns. This table accounts for reflective and rotational symmetries to collapse the standard 256 cases [S1, unofficial].
*   **Intersection Points:** Pre-calculated or computed on-the-fly positions at the exact midpoints of the 12 edges of the cube [S1, unofficial].

**Steps:**
1.  **Grid Generation:** Convert the raw volumetric dataset into a regular 3-D grid. For each logical cube, determine the binary occupancy of its 8 vertices. A vertex is assigned binary 1 if its projection distance on the $i$, $j$, or $k$ axis exceeds the midpoint threshold relative to the surface; otherwise, it is 0 [S1, unofficial].
2.  **Index Calculation:** Construct an 8-bit binary index for the current cube based on the occupancy states of its 8 vertices (vertices #1 to #8) [S1, unofficial].
3.  **Pattern Lookup:** Use the 8-bit index to retrieve the corresponding surface intersection pattern from the reduced lookup table (15 basic patterns) [S1, unofficial].
    *   Pattern 0: No vertices inside (no triangles).
    *   Pattern 1: One vertex inside, etc.
4.  **Intersection Calculation:** For each edge of the cube identified as intersected by the pattern, set the surface intersection point exactly at the midpoint of that edge. No linear interpolation based on scalar values is performed [S1, unofficial].
5.  **Triangle Generation:** Generate triangles using the 12 possible midpoint positions on the cube edges as vertices, following the connectivity defined by the lookup table pattern [S1, unofficial].
6.  **Rendering:** Render the resulting triangle vertex coordinates using OpenGL [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def midpoint_marching_cubes(grid, lookup_table):
    """
    grid: 3D array of binary occupancy values (0 or 1) for each vertex.
    lookup_table: Dictionary mapping 8-bit integer index to a list of triangle definitions.
                  Each triangle definition is a list of 3 edge indices (0-11).
    """
    triangles = []
    # Iterate over all logical cubes in the grid
    # Assuming grid dimensions are (Dx, Dy, Dz)
    for i in range(grid.shape[0] - 1):
        for j in range(grid.shape[1] - 1):
            for k in range(grid.shape[2] - 1):
                # 1. Get the 8 vertices of the current cube
                # Vertices are ordered according to standard MC convention
                v0 = grid[i, j, k]
                v1 = grid[i+1, j, k]
                v2 = grid[i+1, j+1, k]
                v3 = grid[i, j+1, k]
                v4 = grid[i, j, k+1]
                v5 = grid[i+1, j, k+1]
                v6 = grid[i+1, j+1, k+1]
                v7 = grid[i, j+1, k+1]
                
                # 2. Calculate 8-bit index
                # Bit 0 corresponds to v0, Bit 1 to v1, ..., Bit 7 to v7
                index = (v0 << 0) | (v1 << 1) | (v2 << 2) | (v3 << 3) | \
                        (v4 << 4) | (v5 << 5) | (v6 << 6) | (v7 << 7)
                
                # 3. Lookup pattern
                # If index is 0 or 255, no triangles are generated (empty or full cube)
                if index == 0 or index == 255:
                    continue
                
                # Get the list of edges that form triangles for this index
                # The lookup table returns edge indices (0-11) grouped into triangles
                edge_indices = lookup_table[index]
                
                # 4. Generate triangles
                # Process edge indices in groups of 3 to form triangles
                for tri_edges in edge_indices:
                    # tri_edges is a list of 3 edge indices
                    p1 = get_midpoint(i, j, k, tri_edges[0])
                    p2 = get_midpoint(i, j, k, tri_edges[1])
                    p3 = get_midpoint(i, j, k, tri_edges[2])
                    triangles.append([p1, p2, p3])
                    
    return triangles

def get_midpoint(i, j, k, edge_index):
    """
    Returns the midpoint coordinates of a specific edge of the cube at (i,j,k).
    Edge indices 0-11 correspond to the 12 edges of a cube.
    Standard MC edge ordering is assumed.
    """
    # Define the 8 corners of the cube
    corners = [
        (i, j, k),       # 0
        (i+1, j, k),     # 1
        (i+1, j+1, k),   # 2
        (i, j+1, k),     # 3
        (i, j, k+1),     # 4
        (i+1, j, k+1),   # 5
        (i+1, j+1, k+1), # 6
        (i, j+1, k+1)    # 7
    ]
    
    # Define the 12 edges as pairs of vertex indices
    edges = [
        (0, 1), (1, 2), (2, 3), (3, 0), # Bottom face
        (4, 5), (5, 6), (6, 7), (7, 4), # Top face
        (0, 4), (1, 5), (2, 6), (3, 7)  # Vertical edges
    ]
    
    v_start, v_end = edges[edge_index]
    p_start = corners[v_start]
    p_end = corners[v_end]
    
    # Midpoint calculation
    mx = (p_start[0] + p_end[0]) / 2.0
    my = (p_start[1] + p_end[1]) / 2.0
    mz = (p_start[2] + p_end[2]) / 2.0
    
    return (mx, my, mz)
```

**Parameters:**
*   **Threshold for Binary Assignment:** The method relies on a binary classification of vertices. The source notes that a vertex is assigned 1 if its "projection distance on i and j axes exceeds the midpoint" [S1, unofficial]. The exact scalar threshold for this binary conversion is not explicitly defined in the provided text, but it implies a step function at the grid cell center or a specific depth threshold from the scanning system [S1, unofficial].
*   **Lookup Table Size:** 15 basic patterns [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hole Problem:** The method suffers from the "hole problem" because the lookup table depends only on complementary symmetry. This fails to resolve ambiguous cases correctly, potentially leading to topological errors or holes in the mesh [S1, unofficial].
*   **Accuracy Limitations:** High accuracy is not guaranteed for complex geometries. The method is limited to objects with smooth, featureless surfaces [S1, unofficial].
*   **Ambiguity:** Standard MC ambiguities (e.g., the tetrahedron case) are not addressed by midpoint selection alone, as the method relies on the reduced 15-pattern table which may not cover all topological configurations correctly without additional logic [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of cubes in the grid. The lookup is $O(1)$ and midpoint calculation is $O(1)$. No scalar interpolation is required, which is faster than standard MC.
*   **Memory Complexity:** $O(N)$ for storing the grid and the resulting mesh. The lookup table is constant size (15 patterns).
*   **Strengths:** Simple implementation; no need for scalar value interpolation; reduced lookup table size (15 vs 256); fast rendering via OpenGL.
*   **Limitations:** Low accuracy for complex or feature-rich surfaces; prone to the "hole problem" due to symmetry-only lookup; not suitable for high-fidelity reconstruction of detailed geometries [S1, unofficial].
*   **When to use:** Suitable for real-time or resource-constrained applications involving smooth, featureless surfaces (e.g., endoscopic scanning of biological tissues) where speed is prioritized over geometric precision [S1, unofficial].
*   **When not to use:** Not suitable for complex geometries, sharp features, or applications requiring high topological correctness and accuracy [S1, unofficial].

## Background
The method was proposed in the context of 3-D surface reconstruction based on endoscope and optical fiber scanning systems [S1, unofficial]. It was introduced as a "Proposed Marching Cubes Algorithm" to simplify the surface extraction process for volumetric data derived from endoscopic scans [S1, unofficial]. The authors reduced the standard 256 cases to 15 basic patterns using reflective and rotational symmetries, a technique previously studied in other works [S1, unofficial]. The method was evaluated using plane tests and endoscope scanning data, showing limitations in accuracy for non-smooth surfaces [S1, unofficial].

## Sources
[S1] A Marching Cubes Algorithm: Application for Three-dimensional Surface Reconstruction Based on Endoscope and Optical Fiber, https://inria.hal.science/hal-01205823v1, fetched 2026-10-26 (unofficial)
