# Dual Grid Isosurface Extraction

Also called: Dual Grid Marching Cubes, Weber et al. (2001b, 2003a) method

## Overview
Dual Grid Isosurface Extraction extracts crack-free isosurfaces from cell-centered Adaptive Mesh Refinement (AMR) data by treating cell centers as vertices of a new grid [S1]. It solves the dangling node problem and resulting surface discontinuities that occur when standard Marching Cubes is applied to vertex-centered resampled AMR data [S1]. It differs from its siblings by leveraging the natural geometric gap between coarse and fine grid levels in the dual representation to prevent dangling nodes, rather than using post-processing fixes or polygon collapsing [S1, Dual-Grid Non-Manifold Contouring](14936-dual-grid-non-manifold-contouring.md).

## How it works
**Inputs and Outputs:**
*   **Input:** AMR scalar field data defined on cell centers.
*   **Output:** A manifold triangle mesh representing the isosurface.

**Data Structures:**
The method constructs a **dual grid**.
*   **Vertices:** The vertices of the dual grid correspond to the cell centers of the original AMR grid [S1].
*   **Topology:** The dual grid is a structured grid that is "shrunk" by one cell in each axial direction relative to the original grid boundaries [S1]. This creates a geometric gap between coarse and fine grid levels [S1].

**Steps:**
1.  **Construct Dual Grid:** Map each cell center of the AMR grid to a vertex in the dual grid. The scalar value associated with each dual vertex is the original cell-centered value [S1].
2.  **Identify Gap:** Recognize that the dual grid structure inherently creates a gap between the coarse and fine grid regions due to the one-cell shrinkage in each axial direction [S1].
3.  **Extract Isosurface:** Apply the standard Marching Cubes algorithm (as defined in [Grid Structure and Topology](14934-grid-structure-and-topology.md)) on the dual grid vertices. The gap ensures that no dangling nodes exist at the boundaries of refinement levels, preventing cracks [S1].
4.  **Stitching (if required):** If the gap between coarse and fine grids results in uncovered regions, a tessellation scheme (stitch cells) is used to merge the levels seamlessly [S2, unofficial]. This is handled by the variant [Dual Grid Isosurface Extraction with Stitch Cells](10500-dual-grid-isosurface-extraction-with-stitch-cells.md).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard Marching Cubes triangle lookup table (256 cases, 16 entries each, -1 terminator) [G]
# Provided by the parent article [Grid Structure and Topology](14934-grid-structure-and-topology.md)
TRIANGLE_TABLE = [...] 

def extract_isosurface_dual_grid(cubes, isovalue):
    """
    Applies Marching Cubes on the dual grid cubes.
    cubes: List of cubes, each with 'vertices' (list of 8 dicts with 'pos' and 'value').
    Returns: List of triangles, each a tuple of three 3D points.
    """
    triangles = []
    # Standard MC edge-to-vertex mapping (12 edges, each connects two vertices) [G]
    EDGE_VERTICES = [
        (0,1), (1,2), (2,3), (3,0), # Bottom face
        (4,5), (5,6), (6,7), (7,4), # Top face
        (0,4), (1,5), (2,6), (3,7)  # Vertical edges
    ]
    
    for cube in cubes:
        # 1. Compute case index
        case_index = 0
        for i in range(8):
            if cube['vertices'][i]['value'] < isovalue:
                case_index |= (1 << i)
        
        # 2. Compute edge intersections
        edge_intersections = {}
        for edge_idx in range(12):
            v1_idx, v2_idx = EDGE_VERTICES[edge_idx]
            v1 = cube['vertices'][v1_idx]
            v2 = cube['vertices'][v2_idx]
            val1 = v1['value']
            val2 = v2['value']
            # Check if edge is cut
            if (val1 < isovalue) != (val2 < isovalue):
                if abs(val2 - val1) < 1e-8:
                    t = 0.5
                else:
                    t = (isovalue - val1) / (val2 - val1)
                edge_intersections[edge_idx] = (
                    v1['pos'][0] + t * (v2['pos'][0] - v1['pos'][0]),
                    v1['pos'][1] + t * (v2['pos'][1] - v1['pos'][1]),
                    v1['pos'][2] + t * (v2['pos'][2] - v1['pos'][2])
                )
        
        # 3. Generate triangles from lookup table
        tri_indices = TRIANGLE_TABLE[case_index]
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
*   **Time Complexity:** Linear with respect to the number of voxels in the dual grid, similar to standard Marching Cubes [G].
*   **Memory Complexity:** Proportional to the number of cell centers in the AMR data [G].
*   **Strengths:** Eliminates cracks caused by dangling nodes in AMR data without requiring complex post-processing or interpolation artifacts [S1].
*   **Limitations:** Requires a stitching mechanism (like stitch cells) to handle the geometric gaps between coarse and fine levels if a continuous mesh across levels is needed [S2, unofficial].
*   **When to use:** When extracting isosurfaces from AMR data where standard vertex-centered resampling leads to topological errors [S1].

## Background
The method was introduced to address issues in extracting isosurfaces from Adaptive Mesh Refinement (AMR) data [S1]. Standard Marching Cubes on vertex-centered resampled AMR data suffers from dangling nodes, which cause cracks in the extracted surface [S1]. The dual grid approach, defined by cell-centered values, was proposed to solve these problems by leveraging the natural gap between grid levels [S1]. Weber et al. developed the method to extract crack-free isosurfaces by interpreting cell centers as dual grid vertices [S2, unofficial].

## Sources
[S1] Extraction of Crack-free Isosurfaces from Adaptive Mesh Refinement Data, https://doi.org/10.2312/vissym/vissym01/025-034, fetched 2026-10-26
[S2] Visualization of Scalar Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc901373/?q=adaptive mesh refinement visualization ray casting, fetched 2026-10-28 (unofficial)
