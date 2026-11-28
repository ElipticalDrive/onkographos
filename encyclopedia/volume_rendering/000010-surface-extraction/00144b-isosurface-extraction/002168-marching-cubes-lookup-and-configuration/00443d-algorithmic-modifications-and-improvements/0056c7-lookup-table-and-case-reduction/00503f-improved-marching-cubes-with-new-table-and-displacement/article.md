# Improved Marching Cubes with New Table and Displacement

Also called: Improved Macet, MC with displacement edge transform, MC with new table

## Overview
This method improves the worst-case triangle quality of Marching Cubes (MC) and Macet by modifying the lookup table to avoid specific edge groups responsible for poor triangles and applying a non-linear displacement to cut points on edges [S1]. It sets itself apart from standard MC by explicitly optimizing for triangle aspect ratio bounds through edge group analysis, and from Macet by removing neighborhood-based constraints on edge transformations [S1].

## How it works
**Inputs:** A scalar field defined on a structured grid, an isovalue, and the standard Marching Cubes cell configuration (vertex signs) [G].
**Outputs:** A triangular mesh approximating the isosurface [G].

**Data Structures:**
*   **Modified MC Table:** A lookup structure mapping the 256 cell configurations to triangulation indices, reformulated to exclude entries that utilize the specific "bad" edge group identified in the source analysis [S1].
*   **Edge Group Classification:** Triangles are classified by the edge groups they utilize, with one specific group identified as responsible for the worst triangles [S1].

**Parameters:**
*   **Gain Function Parameter ($k$):** Set to $0.3$ [S1].

**Steps:**
1.  **Case Identification:** For each voxel, determine the cell configuration index based on the sign of the scalar field at the 8 vertices relative to the isovalue [G].
2.  **Lookup:** Retrieve the triangulation pattern from the **Modified MC Table**, which is constructed to avoid the edge group associated with poor triangle quality [S1].
3.  **Edge Displacement (Non-linear):** For each edge intersected by the isosurface, calculate the intersection point. Instead of using linear interpolation, apply a gain function to modulate the intersection value [S1]. Let $t$ be the normalized linear interpolation parameter ($0 \le t \le 1$) along the edge. Apply the gain function to displace the cut point [S1]. The source specifies a gain function with parameter $k=0.3$ that increases displacement near endpoints [S1]. The exact formula is not provided in the sources; a standard symmetric power-law implementation is used [G].
4.  **Triangle Assembly:** Construct triangles using the displaced vertices [G].
5.  **Macet Integration (Optional):** If using the Macet variant, apply edge transformations to the triangles. Unlike the original Macet algorithm, this method considers each edge case individually without imposing neighborhood-based constraints [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def improved_marching_cubes(grid, isovalue, gain_param=0.3):
    """
    grid: 3D array of scalar values (shape: nx, ny, nz)
    isovalue: float
    gain_param: float (0.3)
    """
    nx, ny, nz = grid.shape
    triangles = []
    # Pre-define the modified MC table (256 entries, each containing edge indices and triangle indices)
    # The exact entries are precomputed per the source's edge group analysis [S1]
    MODIFIED_MC_TABLE = [None] * 256

    for x in range(nx - 1):
        for y in range(ny - 1):
            for z in range(nz - 1):
                # 1. Determine cell configuration (8 vertices)
                signs = []
                for i in range(8):
                    # Standard MC vertex ordering: bit i corresponds to vertex i
                    v = grid[x + (i & 1), y + ((i >> 1) & 1), z + ((i >> 2) & 1)]
                    signs.append(1 if v >= isovalue else 0)
                
                # Map signs to case index (standard MC indexing)
                case_index = sum(signs[i] << i for i in range(8))
                
                # 2. Lookup from Modified Table
                triangulation = MODIFIED_MC_TABLE[case_index]
                if triangulation is None:
                    continue  # No intersection for this cell
                
                # 3. Calculate displaced edge intersections
                edge_points = {}
                for edge_idx in triangulation.edges:
                    # Standard MC edge indices (0-11)
                    v1_idx, v2_idx = MC_EDGES[edge_idx]
                    v1 = (x + (v1_idx & 1), y + ((v1_idx >> 1) & 1), z + ((v1_idx >> 2) & 1))
                    v2 = (x + (v2_idx & 1), y + ((v2_idx >> 1) & 1), z + ((v2_idx >> 2) & 1))
                    
                    val1 = grid[v1]
                    val2 = grid[v2]
                    
                    # Linear interpolation parameter
                    if abs(val2 - val1) < 1e-9:
                        t = 0.5  # [G] Fallback for degenerate edge
                    else:
                        t = (isovalue - val1) / (val2 - val1)
                    
                    # Apply gain function for non-linear displacement
                    # The source specifies a gain function with parameter 0.3
                    # that increases displacement near endpoints [S1].
                    # The exact formula is not provided in the sources; a standard symmetric power-law is used [G].
                    if t <= 0.5:
                        t_displaced = t ** gain_param
                    else:
                        t_displaced = 1.0 - (1.0 - t) ** gain_param
                    
                    # Interpolate vertex position
                    p = (v1[0] + t_displaced * (v2[0] - v1[0]),
                         v1[1] + t_displaced * (v2[1] - v1[1]),
                         v1[2] + t_displaced * (v2[2] - v1[2]))
                    edge_points[edge_idx] = p
                
                # 4. Assemble Triangles
                for tri in triangulation.triangles:
                    p1 = edge_points[tri.edge1]
                    p2 = edge_points[tri.edge2]
                    p3 = edge_points[tri.edge3]
                    triangles.append([p1, p2, p3])
                    
    return triangles

# Standard MC edge definitions (vertex indices 0-7)
MC_EDGES = [
    (0, 1), (1, 3), (3, 2), (2, 0),  # Bottom face
    (4, 5), (5, 7), (7, 6), (6, 4),  # Top face
    (0, 4), (1, 5), (3, 7), (2, 6)   # Vertical edges
]
```

**Edge Cases and Pitfalls:**
*   **Unavoidable Bad Triangles:** It is not always possible to remove undesirable edge groups from all MC table cases. Some cases (e.g., convex polygons) must always include at least one triangle from the problematic edge group (specifically edge group 2) [S1].
*   **Trade-off:** Improving worst-case triangle quality comes at the expense of reducing the quality of triangles that already have good aspect ratios [S1].
*   **Noise Sensitivity:** In noisy datasets (e.g., Engine), the improvement in worst triangle quality is less substantial compared to cleaner datasets (Silicium, Bonsai, Lobster) [S1].
*   **Max Distance Invariance:** The maximum absolute distance to the isosurface does not change with displacement modulation because the cut point involved in the maximum distance is not affected by the operation [S1].

## Operating characteristics
*   **Triangle Quality:** The method significantly improves the worst-case triangle quality. In three datasets (Silicium, Bonsai, Lobster), the quality of the worst triangle improved by at least an order of magnitude compared to the original MC table [S1].
*   **Comparison to Macet:** The improved Macet with the new table is closer to twice as better in worst triangle quality compared to old Macet with the old table in Bonsai, Lobster, and Silicium datasets [S1].
*   **Quality Bounds:** The minimum quality of transformed edge groups in Macet is 0.272 (edge group 2), corresponding to a triangle with a minimal internal angle of 18 degrees [S1]. In all experiments, the minimum triangle quality was above 0.19 [S1].
*   **Comparison to State-of-the-Art:** Unlike methods by Schreiner et al., Dey and Levine, and Meyer et al., which report worst triangle quality ranging from 0.02 to 0.4, this method maintains a minimum quality above 0.19 [S1].
*   **Accuracy:** The difference in distance from the triangulation to the actual isosurface is nearly the same between the new and old MC tables [S1].
*   **Performance:** The method retains the performance advantages and simplicity of standard Marching Cubes, allowing for acceleration structures like span space methods [S1].
*   **Limitations:** The method is intrinsically based on Marching Cubes and cannot generate meshes that adapt to surface features such as curvature [S1].
*   **Complexity:** Time and memory complexity are not given in the sources.

## Background
*   **Origin:** The method is described in "Edge Groups: An Approach to Understanding the Mesh Quality of Marching Methods" [S1].
*   **Development:** The work identifies that one specific edge group is responsible for most of the worst triangles generated by MC. It proposes a simple reformulation of the MC table entries to avoid this group. It also extends this analysis to Macet, removing neighborhood constraints to apply edge transformations individually [S1].
*   **Context:** Previous works focused on ambiguity resolution (Nielson) or extended tables, but none focused on triangle quality criteria. This work provides quality bounds for triangles generated in each cell, guiding the design of the table [S1].

## Sources
[S1] Edge Groups: An Approach to Understanding the Mesh Quality of Marching Methods, https://doi.org/10.1109/tvcg.2008.122, fetched 2026-10-26
