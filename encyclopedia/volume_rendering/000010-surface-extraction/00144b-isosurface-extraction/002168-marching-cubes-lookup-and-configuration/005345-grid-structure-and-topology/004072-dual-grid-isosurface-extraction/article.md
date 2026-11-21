# Dual Grid Isosurface Extraction

Also called: Dual Grid Marching Cubes

## Overview
Dual Grid Isosurface Extraction is a method for extracting crack-free isosurfaces from Adaptive Mesh Refinement (AMR) data by treating cell centers as vertices of a new grid [S1]. It solves the problem of dangling nodes and resulting surface discontinuities that occur when standard Marching Cubes is applied to vertex-centered resampled AMR data [S1]. It differs from its siblings by leveraging the natural gap between coarse and fine grid levels in the dual representation to prevent dangling nodes, rather than using post-processing fixes or polygon collapsing [S1, Dual-Grid Non-Manifold Contouring](14936-dual-grid-non-manifold-contouring.md).

## How it works
**Inputs and Outputs:**
*   **Input:** AMR scalar field data defined on cell centers.
*   **Output:** A manifold triangle mesh representing the isosurface.

**Data Structures:**
The method constructs a **dual grid**.
*   **Vertices:** The vertices of the dual grid correspond to the cell centers of the original AMR grid.
*   **Topology:** The dual grid is a structured grid that is "shrunk" by one cell in each axial direction relative to the original grid boundaries [S1]. This creates a geometric gap between coarse and fine grid levels.

**Steps:**
1.  **Construct Dual Grid:** Map each cell center of the AMR grid to a vertex in the dual grid. The scalar value associated with each dual vertex is the original cell-centered value [S1].
2.  **Identify Gap:** Recognize that the dual grid structure inherently creates a gap between the coarse and fine grid regions due to the one-cell shrinkage in each axial direction [S1].
3.  **Extract Isosurface:** Apply the standard Marching Cubes algorithm (as defined in [Grid Structure and Topology](14934-grid-structure-and-topology.md)) on the dual grid vertices. The gap ensures that no dangling nodes exist at the boundaries of refinement levels, preventing cracks [S1].
4.  **Stitching (if required):** If the gap between coarse and fine grids results in uncovered regions, a tessellation scheme (stitch cells) is used to merge the levels seamlessly. This is handled by the variant [Dual Grid Isosurface Extraction with Stitch Cells](10500-dual-grid-isosurface-extraction-with-stitch-cells.md).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_dual_grid(AMR_cells):
    """
    Maps AMR cell centers to dual grid vertices.
    AMR_cells: List of cells, each with center coordinates (x, y, z) and value v.
    Returns: A structured grid representation where vertices are cell centers.
    """
    # The dual grid is defined by the function values at the cell centers.
    # The grid shrinks by one cell in each axial direction relative to the original grid.
    # This creates a gap between coarse and fine grids.
    dual_vertices = []
    for cell in AMR_cells:
        dual_vertices.append({
            'pos': cell.center,
            'value': cell.value
        })
    return dual_vertices

def extract_isosurface_dual_grid(dual_vertices, isovalue):
    """
    Applies Marching Cubes on the dual grid.
    """
    # Since the dual grid handles the topology via the gap,
    # we can use the standard Marching Cubes logic on the dual vertices.
    # The key is that the dual grid structure prevents dangling nodes.
    
    # 1. Build the voxel structure from dual vertices
    # Note: The exact voxelization depends on the AMR hierarchy structure.
    # For a uniform dual grid representation:
    voxels = build_voxels_from_dual_vertices(dual_vertices)
    
    triangles = []
    for voxel in voxels:
        # Get values at the 8 corners of the voxel in the dual grid
        values = [vertex['value'] for vertex in voxel.corners]
        
        # Compute case index
        case_index = 0
        for i in range(8):
            if values[i] < isovalue:
                case_index |= (1 << i)
        
        # Look up edge intersections and triangles
        # edge_table and triangle_table are from [Grid Structure and Topology](14934-grid-structure-and-topology.md)
        edge_mask = edge_table[case_index]
        tri_indices = triangle_table[case_index]
        
        # Calculate intersection points
        edge_intersections = {}
        for edge_idx in range(12):
            if (edge_mask >> edge_idx) & 1:
                # Interpolate on the edge of the dual grid voxel
                p1 = voxel.corners[edge_start[edge_idx]]
                p2 = voxel.corners[edge_end[edge_idx]]
                v1 = p1['value']
                v2 = p2['value']
                # Linear interpolation
                t = (isovalue - v1) / (v2 - v1)
                pos = lerp(p1['pos'], p2['pos'], t)
                edge_intersections[edge_idx] = pos
        
        # Generate triangles
        for i in range(0, len(tri_indices), 3):
            if tri_indices[i] == -1:
                break
            e1, e2, e3 = tri_indices[i], tri_indices[i+1], tri_indices[i+2]
            p1 = edge_intersections[e1]
            p2 = edge_intersections[e2]
            p3 = edge_intersections[e3]
            triangles.append((p1, p2, p3))
            
    return triangles
```

## Subtopics

### [Dual Grid Isosurface Extraction with Stitch Cells](10500-dual-grid-isosurface-extraction-with-stitch-cells.md)
Extracts continuous, crack-free isosurfaces from AMR data by constructing a dual grid that connects cell centers, thereby eliminating dangling nodes inherent in standard Marching Cubes on vertex-centered resampled data. It differs from previous crack-fixing solutions that create polygons, collapse polylines, or use triangle fans by using a dual grid and tessellated "stitch cells" to maintain topological consistency without interpolation artifacts. Choose this when seamless merging of AMR levels is required without manual gap handling.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the dual grid, similar to standard Marching Cubes.
*   **Memory Complexity:** Proportional to the number of cell centers in the AMR data.
*   **Strengths:** Eliminates cracks caused by dangling nodes in AMR data without requiring complex post-processing or interpolation artifacts.
*   **Limitations:** Requires a stitching mechanism (like stitch cells) to handle the geometric gaps between coarse and fine levels if a continuous mesh across levels is needed.
*   **When to use:** When extracting isosurfaces from AMR data where standard vertex-centered resampling leads to topological errors.

## Background
The method was introduced to address issues in extracting isosurfaces from Adaptive Mesh Refinement (AMR) data. Standard Marching Cubes on vertex-centered resampled AMR data suffers from dangling nodes, which cause cracks in the extracted surface [S1]. The dual grid approach, defined by cell-centered values, was proposed to solve these problems by leveraging the natural gap between grid levels [S1].

## Sources
[S1] Extraction of Crack-free Isosurfaces from Adaptive Mesh Refinement Data, https://doi.org/10.2312/vissym/vissym01/025-034, fetched 2026-10-26
