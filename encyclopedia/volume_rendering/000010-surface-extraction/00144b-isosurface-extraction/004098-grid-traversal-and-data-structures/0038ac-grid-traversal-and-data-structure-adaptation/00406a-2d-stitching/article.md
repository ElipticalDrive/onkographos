# 2D Stitching
Also called: Stitching 2D Grids

## Overview
2D Stitching is an index-based tessellation scheme that fills topological gaps between dual grids of different Adaptive Mesh Refinement (AMR) levels using "stitch cells," allowing seamless merging of levels without subdividing boundary elements [S1]. It is distinguished from constrained Delaunay triangulation by avoiding the subdivision of quadrilateral boundary faces, which is non-unique and causes cracks in 3D Marching Cubes extraction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Two adjacent 2D AMR grids (a coarse grid $\Lambda_{coarse}$ and a fine grid $\Lambda_{fine}$) with refinement ratio $r$, represented by their dual grid vertices and edges [S1].
*   **Output:** A set of stitch cells (triangles and deformed quadrilaterals) that connect the boundary of the fine grid to the coarse grid [S1].

**Data Structures:**
The method operates on the dual grid representation [S1].
*   **Vertices:** Defined by their grid coordinates and boundary status [S1].
*   **Edges:** Defined by pairs of vertices [S1].
*   **Stitch Cells:** Geometric primitives (triangles, quads) defined by the vertex indices they connect; no new vertices are generated, only existing vertices from the coarse and fine grids are used [S1].

**Steps:**
1.  Identify the boundary edges and vertices of the fine grid adjacent to the coarse grid [S1].
2.  For each boundary edge, determine the corresponding coarse grid location based on the refinement ratio $r$ [S1].
3.  Generate edge stitch cells by connecting fine edges to coarse points or edges according to $r$ and the refinement status of the coarse grid points [S1].
4.  For each boundary vertex, generate vertex stitch cells by connecting it to the coarse grid via two triangles [S1].
5.  Assemble the stitch cells with the original dual grid elements into a single mesh [S1].

**Formulas and Logic:**
**Refinement Ratio:**
Let $r$ be the integer refinement ratio ($r \ge 2$). $r$ fine grid cells fit into one coarse grid cell [S1].

**Edge Connections (General $r$):**
For a boundary edge of the fine grid, the connection to the coarse grid depends on $r$:
*   Connect $(r-1)$ consecutive fine edge segments to one common coarse-grid point [S1].
*   Connect every $r$-th fine edge to a coarse edge [S1].

**Edge Connections ($r=2$ Specifics):**
For $r=2$, each fine edge is connected alternatingly to either a vertex or an edge of the coarse grid [S1]. The specific topology depends on whether the coarse grid points adjacent to the connection are refined:
*   If a fine edge connects to a coarse point that is refined, connect it to another fine edge to form a rectilinear cell (quadrilateral) [S1].
*   If both endpoints of a coarse edge are refined, connect the fine edge to another fine edge [S1].
*   If only one endpoint of a coarse edge is refined, apply the same tessellation as other edge cases to maintain consistency [S1].

**Vertex Connections:**
For a boundary vertex $v$ of the fine grid:
*   Connect the vertex to the coarse grid via two triangles [S1].
*   Connect each edge of the vertex to the two coarse edges that are "visible" from it [S1].
*   If a coarse grid point is refined, promote the border vertex to a border-edge segment by connecting it to the other refined grid point, then apply the edge connection logic [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_stitch_cells(fine_edges, fine_vertices, coarse_grid, r):
    """
    Generates stitch cells for the boundary between a coarse and fine AMR grid.
    Args:
        fine_edges: List of edge objects with attributes .v1, .v2, .is_boundary
        fine_vertices: List of vertex objects with attributes .idx, .is_boundary, .get_adjacent_edges()
        coarse_grid: Object with methods .is_refined(idx), .get_coarse_edge(idx), .get_visible_coarse_edges(edge, idx)
        r: Integer refinement ratio
    Returns:
        List of stitch cells (each cell is a list of vertex indices)
    """
    stitch_cells = []

    # Process boundary edges
    for edge in fine_edges:
        if not edge.is_boundary:
            continue
        v1, v2 = edge.v1, edge.v2
        coarse_idx = v1 // r  # Index mapping for regular grid structure
        coarse_point_refined = coarse_grid.is_refined(coarse_idx)

        if coarse_point_refined:
            # Connect to another fine edge to form a rectilinear cell
            adj_edge = find_adjacent_boundary_edge(fine_edges, edge)
            if adj_edge:
                v3, v4 = adj_edge.v1, adj_edge.v2
                stitch_cells.append([v1, v2, v4, v3])
        else:
            # Connect to coarse point or edge based on r
            edge_idx = fine_edges.index(edge)
            if edge_idx % r == 0:
                # Connect to coarse edge
                coarse_edge = coarse_grid.get_coarse_edge(coarse_idx)
                c1, c2 = coarse_edge.v1, coarse_edge.v2
                stitch_cells.append([v1, v2, c2, c1])
            else:
                # Connect to coarse point
                stitch_cells.append([v1, v2, coarse_idx])

    # Process boundary vertices
    for vertex in fine_vertices:
        if not vertex.is_boundary:
            continue
        v_idx = vertex.idx
        coarse_vertex_idx = v_idx // r
        coarse_vertex_refined = coarse_grid.is_refined(coarse_vertex_idx)

        if coarse_vertex_refined:
            # Promote to border-edge segment
            adj_refined = find_adjacent_refined_coarse_point(coarse_grid, coarse_vertex_idx)
            if adj_refined:
                # Apply edge connection logic to the promoted segment
                # stitch_cells.append(...) using the edge logic above
                pass
        else:
            # Connect via two triangles to visible coarse edges
            edges_at_vertex = vertex.get_adjacent_edges()
            for edge in edges_at_vertex:
                visible_coarse_edges = coarse_grid.get_visible_coarse_edges(edge, coarse_vertex_idx)
                for c_edge in visible_coarse_edges:
                    c1, c2 = c_edge.v1, c_edge.v2
                    stitch_cells.append([v_idx, edge.v2, c1])
                    stitch_cells.append([v_idx, c1, c2])

    return stitch_cells

def find_adjacent_boundary_edge(edges, current_edge):
    for e in edges:
        if e is not current_edge and (e.v1 == current_edge.v2 or e.v2 == current_edge.v1):
            return e
    return None

def find_adjacent_refined_coarse_point(coarse_grid, idx):
    for neighbor in coarse_grid.get_neighbors(idx):
        if coarse_grid.is_refined(neighbor):
            return neighbor
    return None
```

**Parameters:**
*   **Refinement ratio $r$:** An integer specifying how many fine grid cells fit into a coarse grid cell. Typical values are 2 and 4 [S1].

**Edge Cases and Pitfalls:**
*   **Quadrilateral Subdivision:** Quadrilaterals formed by stitching must not be subdivided, as subdivision is not unique and causes cracks in 3D [S1].
*   **Mixed Refinement:** Cases where only one endpoint of a coarse edge is refined must be handled consistently with other edge cases to avoid topological errors [S1].

## Subtopics

### [3D AMR Grid Stitching](10250-3d-amr-grid-stitching.md)
Generalizes the 2D index-based stitching to 3D volumes by constructing intermediate "stitch cells" (pyramids, deformed prisms, hexahedra) to connect fine grid vertices/edges to the coarse grid. It handles the increased complexity of 3D vertex and edge interactions.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Ensures crack-free isosurfaces in 3D by avoiding non-unique quadrilateral subdivision [S1].
*   **Limitations:** Requires dual grid representation and integer refinement ratios [S1].
*   **When to use:** When extracting isosurfaces from AMR data where crack-free connectivity between different refinement levels is critical [S1].
*   **When not to use:** When the grid structure is not hierarchical AMR or when non-integer refinement ratios are required [S1].

## Background
The method was introduced in the context of extracting crack-free isosurfaces from Adaptive Mesh Refinement (AMR) data [S1]. It addresses the problem of gaps between dual grids of different AMR levels that lead to discontinuities in the extracted isosurface [S1]. The approach was chosen over constrained Delaunay triangulation because the latter requires subdividing quadrilateral boundary faces, which is problematic for 3D Marching Cubes extraction [S1].

## Sources
[S1] Extraction of Crack-free Isosurfaces from Adaptive Mesh Refinement Data, https://doi.org/10.2312/vissym/vissym01/025-034, fetched 2026-10-26
