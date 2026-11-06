# Marching Tetrahedra 3 (DLC-Plane)

Also called: 5-Tetrahedron Cell, DLC-Plane, Diamond Crystal Lattice Tetrahedralization, Diamond Lattice Mesh, Dual Marching Tetrahedra - Diamond Lattice Crystal (DMT-DLC)

## Overview
Marching Tetrahedra 3 (DLC-Plane) is a volume meshing method that generates isosurfaces from voxel or point cloud data using a diamond crystal lattice structure [S1, S2, unofficial]. It decomposes the grid into cells containing exactly 5 tetrahedra, eliminating the need for fabricated center points [S1, S2, unofficial]. Unlike standard Marching Cubes or earlier Marching Tetrahedra variants that require synthesizing internal vertices, this method uses only the 5 existing grid points per cell, reducing boundary creases and improving results for opening void areas [S1, S2, unofficial]. It operates by slicing the data plane-by-plane, generating a single calculation for all shared points to reduce total point count and computational overhead [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A scalar field defined on a 3D grid or point cloud.
- An isosurface threshold value (typically 0).
- A mode flag: `DLC-Plane` (plane-by-plane processing) or `DLC-Volume` (cloud-by-cloud processing). This article details `DLC-Plane`.
- A boolean flag `zero_as_inside`: If true, values $\le 0$ are considered inside; if false, values $> 0$ are inside. For distance-to-surface point clouds where density is defined as $-distance$, this must be enabled [S1, S2, unofficial].

**Outputs:**
- A triangle mesh representing the isosurface.
- Vertex positions, normals, and optional texture coordinates.

### Data Structures
The method processes cells based on a diamond cubic lattice. Each cell is defined by 5 vertices. The vertices are indexed to facilitate a specific ordering where every other point is inverted, allowing modulo arithmetic for indexing [S2, unofficial].

The cell geometry is mapped using 5 tetrahedra (indexed 0–4). The tetrahedra are defined by specific vertex indices from the local cell and potentially adjacent cells. The vertex indexing for centroid calculations and tetrahedron definition requires a structured access pattern. The sources describe a mapping for averaging indices that implies a 3x3x2 matrix of values for the underlying grid, but the cell itself only references 5 points [S1, unofficial].

The 5 tetrahedra are defined by the following vertex indices (using local cell indices 0–4 and potentially adjacent indices 5–8 for connectivity, though the core cell is 5 points):
1.  Tetrahedron 0: Vertices 0, 1, 3, 4
2.  Tetrahedron 1: Vertices 1, 2, 4, 5
3.  Tetrahedron 2: Vertices 3, 4, 6, 7
4.  Tetrahedron 3: Vertices 4, 5, 7, 8
5.  Tetrahedron 4: (Implicitly defined by the remaining connectivity or the central structure, though the source explicitly lists 4 tetrahedra with indices up to 8, it states "5 tetrahedra" and "no fabricated points". The 5th tetrahedron completes the diamond lattice cell. Based on the diamond lattice structure, the 5th tetrahedron typically connects the central point to the remaining faces. However, the source explicitly lists the mapping for 4 tetrahedra and mentions "5 tetrahedra". We assume the 5th tetrahedron is the central one or the complement. Given the "alternating same-cells" description, the structure is periodic. The source lists: `0134, 1245, 3467, 4578`. This covers 5 unique vertices (0,1,2,3,4,5,6,7,8) if we consider the cell spans multiple grid points. However, the text says "requires only the 5 points of the cell". This implies the 5 points are the vertices of the diamond lattice unit cell. The standard diamond lattice unit cell has 8 atoms, but the "5-tetrahedron" decomposition likely refers to a specific subset or a simplified cell. Let us stick to the explicit tetrahedra listed: `0134`, `1245`, `3467`, `4578`. This involves 9 indices. This contradicts "5 points". Re-reading: "This only requires the 5 points of the cell much like yours." and "The tetrahedra are indexed 0-4... 0134, 1245...". It is likely that indices 0-4 are the local cell vertices, and the other indices refer to neighbors. However, for a self-contained implementation, we must define the cell. The "Diamond Crystal Lattice" cell in this context is likely a rhombic dodecahedron or similar, decomposed into 5 tetrahedra. The source explicitly states "5 tetrahedra with no fabricated points, requiring only the 5 points of the cell". This suggests the 5 points are the vertices of the cell. The indices `0134` etc. might be referring to a larger grid indexing scheme. For the purpose of the algorithm, we define a cell by 5 vertices $V_0, V_1, V_2, V_3, V_4$. The tetrahedra are formed by subsets of these. The source lists `0134` and `1245`. If the cell has 5 vertices, indices 0-4, then `0134` uses $V_0, V_1, V_3, V_4$. `1245` uses $V_1, V_2, V_4, V_5$. This implies $V_5$ is a neighbor. The "alternating same-cells" suggests a tiling. We will define the cell vertices as $P_0, \dots, P_4$. The tetrahedra are:
    - $T_0: (P_0, P_1, P_3, P_4)$
    - $T_1: (P_1, P_2, P_4, P_{neighbor})$ -> This requires neighbor data.
    
    Actually, the source says "The tetrahedra are indexed 0-4. Each tetrahedron is defined by specific vertex indices from the local cell and potentially adjacent cells". This means the algorithm must handle inter-cell connectivity. However, the "5 points of the cell" statement is key. In a diamond lattice, the primitive cell contains 2 atoms. The "5-tetrahedron" cell is likely a specific decomposition of a cube or rhombic dodecahedron. Given the ambiguity of the indices `0134` etc. which go up to 8, and the statement "5 points", it is highly probable that the "5 points" refers to the *unique* vertices in the local cell, and the indices are global or relative. For implementation, we assume the standard Marching Tetrahedra approach: decompose the cube into 5 tetrahedra. The "Diamond Lattice" here refers to the *orientation* of the tetrahedra. The source mentions "alternating same-cells where the inside tetrahedron is rotated 90 degrees". This implies a checkerboard pattern of tetrahedra orientations.

    Let us define the cell vertices $V_0, \dots, V_7$ of a cube, but the method uses a specific subset or decomposition. The source says "replaces standard marching tetrahedra cells with alternating same-cells". Standard MT uses 5 or 6 tetrahedra per cube. This method uses 5. The vertices are the 8 cube corners, but the "5 points" comment is confusing. It might mean 5 *types* of points or a reduced set. However, "no fabricated points" means no center point. Standard MT also has no fabricated points. The difference is the decomposition. The diamond lattice decomposition of a cube into 5 tetrahedra is not standard (usually 6). The "5-tetrahedron cell" might refer to a rhombic dodecahedron cell in a BCC lattice.

    Given the constraint "implement from plain text", and the source's explicit list of tetrahedra `0134, 1245, 3467, 4578`, we must assume a grid indexing scheme. The source mentions a "3x3x2 matrix of values". This suggests the cell spans multiple grid points.

    To resolve this for implementation: The algorithm iterates through grid planes. For each plane, it processes cells. The cell is defined by 5 vertices. The tetrahedra are formed by these 5 vertices. The specific connectivity is:
    - $T_0: (0, 1, 3, 4)$
    - $T_1: (1, 2, 4, 5)$
    - $T_2: (3, 4, 6, 7)$
    - $T_3: (4, 5, 7, 8)$
    - $T_4: (0, 2, 6, 8)$? (To close the volume). The source says "5 tetrahedra". The listed 4 involve indices 0-8. This is 9 points. This contradicts "5 points".
    
    Correction: The source says "This only requires the 5 points of the cell much like yours." referring to the reader's previous article. The "5 points" likely refers to the fact that the diamond lattice unit cell (rhombic dodecahedron) can be decomposed into 5 tetrahedra using only its vertices. The indices 0-8 in the source might be from a specific test case or a larger stencil. For a general implementation, we define the cell by its 5 vertices $P_0, \dots, P_4$. The tetrahedra are:
    1. $(P_0, P_1, P_2, P_3)$
    2. $(P_0, P_1, P_3, P_4)$
    3. $(P_0, P_2, P_3, P_4)$
    4. $(P_1, P_2, P_3, P_4)$
    5. $(P_0, P_1, P_2, P_4)$?
    
    Actually, a rhombic dodecahedron has 14 vertices. A "5-tetrahedron cell" is a specific decomposition of a cube into 5 tetrahedra? No, a cube is 6.
    
    Let's rely on the "Diamond Crystal Lattice" description. The diamond lattice is a FCC lattice with a 2-atom basis. The "5-tetrahedron" method likely refers to the decomposition of the Wigner-Seitz cell or a similar primitive cell.
    
    Given the ambiguity, the safest implementation path described in the notes is:
    1. Iterate planes.
    2. For each cell, identify 5 vertices.
    3. Decompose into 5 tetrahedra.
    4. Compute intersections.
    
    The source explicitly lists the tetrahedra indices `0134, 1245, 3467, 4578`. This looks like a specific indexing for a 3x3x2 block. We will assume the cell is defined by 5 vertices $V_0..V_4$ and the tetrahedra are formed by subsets. The "alternating" nature means the orientation flips.

    **Vertex Ordering:**
    Vertices are ordered such that every other point is inverted. This allows a `mod 4` operation for indexing [S2, unofficial].
    
    **Sidedness:**
    Sidedness is determined by comparing scalar values to the threshold (0).
    - If `zero_as_inside` is false: Values $\le 0$ are Outside, $> 0$ are Inside.
    - If `zero_as_inside` is true: Values $> 0$ are Outside, $\le 0$ are Inside.
    
    **Edge Intersections:**
    For each edge of a tetrahedron, check if it crosses the isosurface.
    Calculate the delta $T$ along the line from one point to another.
    If a vertex is exactly 0, the state does not change from Outside to Inside (if 0 is Outside) or Inside to Outside (if 0 is Inside).
    If the first point is 0 (and 0 is Outside), no inversion takes place.
    If points 1, 2, or 3 are 0, the slope delta is forced to 0, making the 0 'inside' point as close to itself as possible [S1, S2, unofficial].

### Numbered Steps
1.  **Initialize:** Set up the scalar field and threshold. Determine `zero_as_inside` flag.
2.  **Iterate Planes:** Loop through each plane of the 3D grid (e.g., along the Z-axis).
3.  **Process Cells:** For each cell in the current plane:
    a.  **Identify Vertices:** Retrieve the 5 vertices of the diamond lattice cell. Apply the alternating inversion ordering.
    b.  **Determine Sidedness:** For each vertex, determine if it is Inside or Outside based on the scalar value and threshold.
    c.  **Handle Zero Values:** If a vertex value is exactly 0, treat it consistently with the `zero_as_inside` flag. If 0 is Outside, it is Outside. If 0 is Inside, it is Inside. If a vertex is 0 and is an "inside" point, force the slope delta to 0 to keep it close to itself [S1, S2, unofficial].
    d.  **Decompose into Tetrahedra:** Split the cell into 5 tetrahedra using the diamond lattice connectivity.
    e.  **Compute Intersections:** For each tetrahedron:
        i.  For each edge, check if the endpoints have different sidedness.
        ii. If they do, compute the intersection point using linear interpolation.
        iii. If an endpoint is 0, use the forced delta logic [S1, S2, unofficial].
    f.  **Emit Triangles:** Generate triangles from the intersection points. Emit triangles, not quads, to avoid inverted triangles due to boundary conditions [S1, unofficial].
    g.  **Compute Normals:** All triangles are trivially aligned with their proper normals [S1, unofficial].
4.  **Merge Points:** In `DLC-Plane` mode, shared points between planes are calculated once and reused, reducing point count [S1, S2, unofficial].
5.  **Output:** Return the triangle mesh.

### Formulas
**Linear Interpolation for Intersection:**
Given two vertices $A$ and $B$ with scalar values $S_A$ and $S_B$, and threshold $T=0$.
If $S_A$ and $S_B$ have different signs (or one is 0 and handled appropriately), the intersection point $P$ is:
$$ P = A + \frac{0 - S_A}{S_B - S_A} (B - A) $$
If $S_A = 0$ and $S_B \neq 0$, and 0 is Outside, then $A$ is Outside. The intersection is at $A$ if $B$ is Inside? No, if $A$ is 0 (Outside) and $B$ is Inside, the intersection is at $A$? The source says "if points 1, 2, or 3 are 0, the slope delta is forced to 0, making the 0 'inside' point as close to itself as possible". This implies if the "inside" point is 0, the intersection is at that point.

**Sidedness Check:**
$$ \text{Inside}(V) = \begin{cases} S_V \le 0 & \text{if } \text{zero\_as\_inside} \\ S_V > 0 & \text{if } \neg \text{zero\_as\_inside} \end{cases} $$

### Parameters
- **Threshold:** 0.
- **Zero as Inside:** Boolean. Default false for standard density, true for distance-to-surface point clouds where density = -distance [S1, S2, unofficial].
- **Mode:** `DLC-Plane` or `DLC-Volume`. `DLC-Plane` is recommended for reducing point count [S1, S2, unofficial].

### Edge Cases and Pitfalls
- **Zero Values:** If a vertex is exactly 0, the state transition must be handled carefully. If 0 is Outside, it is Outside. If 0 is Inside, it is Inside. The slope delta is forced to 0 for 0-valued "inside" points to prevent numerical instability [S1, S2, unofficial].
- **Boundary Conditions:** Quad emission can result in inverted triangles. Always emit triangles [S1, unofficial].
- **Vertex Ordering:** Incorrect vertex ordering can lead to incorrect "outside" point detection. Use the alternating inversion ordering with `mod 4` wrapping [S2, unofficial].
- **Fabricated Points:** Do not synthesize center points. Use only the 5 existing vertices [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_tetrahedra_3_dlc_plane(grid, threshold=0.0, zero_as_inside=False):
    """
    Marching Tetrahedra 3 (DLC-Plane) implementation.
    
    Args:
        grid: 3D array of scalar values.
        threshold: Isosurface threshold.
        zero_as_inside: If True, values <= threshold are inside.
        
    Returns:
        List of triangles (each triangle is a list of 3 vertices).
    """
    triangles = []
    visited_points = {} # To handle shared points in DLC-Plane mode
    
    # Iterate through planes (e.g., Z-axis)
    for z in range(grid.shape[0] - 1):
        for y in range(grid.shape[1] - 1):
            for x in range(grid.shape[2] - 1):
                # Get the 5 vertices of the diamond lattice cell
                # Note: The exact indexing depends on the lattice orientation.
                # Assuming a standard decomposition where 5 vertices are used.
                # The source mentions indices 0-4 for the cell.
                # We assume the grid provides these 5 points.
                v0 = get_vertex(grid, x, y, z, 0)
                v1 = get_vertex(grid, x, y, z, 1)
                v2 = get_vertex(grid, x, y, z, 2)
                v3 = get_vertex(grid, x, y, z, 3)
                v4 = get_vertex(grid, x, y, z, 4)
                
                vertices = [v0, v1, v2, v3, v4]
                
                # Determine sidedness
                sidedness = []
                for v in vertices:
                    val = v.scalar
                    if zero_as_inside:
                        sidedness.append(val <= threshold)
                    else:
                        sidedness.append(val > threshold)
                
                # Define the 5 tetrahedra
                # Based on source: 0134, 1245, 3467, 4578...
                # This implies a larger stencil. For a self-contained cell,
                # we use the 5 vertices.
                # Standard 5-tetrahedron decomposition of a cube is not unique.
                # The diamond lattice decomposition is specific.
                # We assume the following connectivity for the 5 vertices:
                tetrahedra = [
                    (0, 1, 3, 4),
                    (0, 1, 2, 3),
                    (0, 2, 3, 4),
                    (1, 2, 3, 4),
                    (0, 1, 2, 4) # Placeholder for the 5th, source is ambiguous
                ]
                
                for tet in tetrahedra:
                    # Get vertices for this tetrahedron
                    t_verts = [vertices[i] for i in tet]
                    t_sidedness = [sidedness[i] for i in tet]
                    
                    # Compute intersections for each edge
                    edges = [
                        (0, 1), (0, 2), (0, 3),
                        (1, 2), (1, 3),
                        (2, 3)
                    ]
                    
                    intersections = []
                    for e in edges:
                        i1, i2 = e
                        if t_sidedness[i1] != t_sidedness[i2]:
                            # Check for zero handling
                            val1 = t_verts[i1].scalar
                            val2 = t_verts[i2].scalar
                            
                            # If one is 0, handle delta
                            if val1 == 0 or val2 == 0:
                                # Force delta to 0 if the "inside" point is 0
                                # This is a simplification of the source's logic
                                if (zero_as_inside and val1 == 0) or (not zero_as_inside and val1 == 0):
                                    # val1 is the "inside" 0 point
                                    intersection = t_verts[i1].position
                                elif (zero_as_inside and val2 == 0) or (not zero_as_inside and val2 == 0):
                                    intersection = t_verts[i2].position
                                else:
                                    # Standard interpolation
                                    t = (threshold - val1) / (val2 - val1)
                                    intersection = lerp(t_verts[i1].position, t_verts[i2].position, t)
                            else:
                                t = (threshold - val1) / (val2 - val1)
                                intersection = lerp(t_verts[i1].position, t_verts[i2].position, t)
                            
                            intersections.append(intersection)
                    
                    # Emit triangles
                    # The source says "emit triangles, not quads"
                    # The number of triangles depends on the configuration
                    # Standard MT emits 0, 1, or 2 triangles per tetrahedron
                    # Here, we assume a standard configuration mapping
                    # This part is ambiguous in the source, so we use a generic MT approach
                    # for the 5-tetrahedron cell.
                    emit_triangles_from_tetrahedron(intersections, tet, triangles)

    return triangles

def get_vertex(grid, x, y, z, index):
    """
    Retrieve the 5 vertices of the diamond lattice cell at (x,y,z).
    This is a placeholder for the specific lattice indexing.
    """
    # Implementation depends on the specific diamond lattice orientation
    # The source mentions "alternating same-cells"
    pass

def lerp(a, b, t):
    return a + t * (b - a)

def emit_triangles_from_tetrahedron(intersections, tet_indices, triangles):
    """
    Emit triangles based on the intersection points.
    The source does not specify the exact triangle configuration for the 5-tetrahedron cell.
    Standard MT uses a lookup table.
    """
    # Placeholder for the specific triangle emission logic
    pass
```

## Subtopics

### [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md)
A variant that regularizes the tetrahedral decomposition to reduce artifacts. Choose when smoothness is critical.

### [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md)
Analyzes the geometric errors of MT methods. Choose for theoretical understanding.

### [Subgrid Pipeline](1640-subgrid-pipeline.md)
A pipeline for processing subgrids. Choose for large-scale data.

### [McGrids](2289-mcgrids.md)
A grid-based approach for Marching Cubes. Choose for cube-based decomposition.

### [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md)
Uses a Body-Centered Cubic lattice. Choose for BCC grids.

### [Exact Fiber Surface Extraction](3445-exact-fiber-surface-extraction.md)
Specialized for fiber structures. Choose for fiber data.

### [Marching Tetrahedra Reconstruction](3516-marching-tetrahedra-reconstruction.md)
Focuses on reconstruction quality. Choose for high-fidelity surfaces.

### [Raster Algorithm for Fiber Surface Extraction](4960-raster-algorithm-for-fiber-surface-extraction.md)
Raster-based approach. Choose for GPU acceleration.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of cells. The `DLC-Plane` mode reduces redundant calculations by sharing points between planes [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ for the output mesh. The `DLC-Plane` mode reduces the total point count compared to `DLC-Volume` [S1, S2, unofficial].
- **Performance:** The method yields good results for opening spaces without discontinuities [S2, unofficial]. It is simpler than the cuboctohedral approach which requires 14 tetrahedra and pyramidal penta-hedrons [S1, unofficial].
- **Strengths:** No fabricated points, reduced boundary creases, code simplicity, good for opening voids [S1, S2, unofficial].
- **Limitations:** Ambiguity in the exact tetrahedron connectivity for the 5-point cell. Requires careful handling of zero values [S1, S2, unofficial].
- **When to use:** When using diamond lattice structures, when avoiding fabricated points is critical, and for opening void areas [S1, S2, unofficial].
- **When not to use:** When standard cube-based decomposition is sufficient, or when the grid is not aligned with the diamond lattice [S1, S2, unofficial].

## Background
The method was developed by d3x0r and is described in the repositories `MarchingTetrahedra` and `DualMarchingTetrahedra` [S1, S2, unofficial]. It is inspired by the diamond crystal lattice structure [S1, S2, unofficial]. The method was designed to improve upon the cuboctohedral approach, which caused creases and indentations on boundaries due to synthesized center points [S1, S2, unofficial]. It is used for flat isosurface mesh calculation with added texturing, demonstrating 3D volume repeating textures [S2, unofficial].

## Sources
[S1] d3x0r/MarchingTetrahedra, https://github.com/d3x0r/MarchingTetrahedra, fetched 2026-10-27 (unofficial)
[S2] d3x0r/DualMarchingTetrahedra, https://github.com/d3x0r/DualMarchingTetrahedra, fetched 2026-10-27 (unofficial)
