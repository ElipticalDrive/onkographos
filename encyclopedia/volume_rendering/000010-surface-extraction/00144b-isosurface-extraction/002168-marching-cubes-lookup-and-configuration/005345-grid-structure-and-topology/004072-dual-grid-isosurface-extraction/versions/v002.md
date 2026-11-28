# Dual Grid Isosurface Extraction
Also called: Dual Grid Marching Cubes
## Overview
Dual Grid Isosurface Extraction extracts continuous, crack-free isosurfaces from cell-centered Adaptive Mesh Refinement (AMR) data by treating cell centers as vertices of a new grid, thereby avoiding the dangling nodes inherent in standard Marching Cubes on vertex-centered resampled data [S1]. It differs from previous crack-fixing solutions that create polygons, collapse polylines, or use triangle fans by using a dual grid and tessellated "stitch cells" to maintain topological consistency without interpolation artifacts [S1].
## How it works
**Inputs:** AMR grid hierarchy with cell-centered scalar field values.
**Outputs:** Triangle mesh representing the isosurface.

**Data Structures:**
- **Dual Grid:** A vertex-centered grid where each vertex corresponds to the center of a cell in the original AMR grid.
- **Stitch Cells:** Tessellated polygons used to bridge gaps between coarse and fine dual grids.

**Steps:**
1. **Construct Dual Grid:** Map each cell center of the AMR grid to a vertex in the dual grid. The scalar value at each dual vertex is the cell-centered value from the original grid [S1].
2. **Identify Gap:** Recognize that the dual grid shrinks by one cell in each axial direction relative to the original grid boundaries. This creates a geometric gap between coarse and fine grid levels [S1].
3. **Extract Isosurface:** Apply the Marching Cubes algorithm on the dual grid vertices. The existence of the gap prevents dangling nodes, which are the primary cause of cracks in standard MC on resampled data [S1].
4. **Stitch Levels:** Use a tessellation scheme to "stitch" grids of different hierarchy levels across the gap. The stitch mesh is constrained by the boundaries of the coarse and fine grids to merge levels seamlessly [S1].

**Edge Cases:**
- **Dangling Nodes:** Standard MC on vertex-centered resampled data suffers from dangling nodes where fine-level vertices lack coarse-level support. The dual grid approach eliminates this by design [S1].
- **Gap Management:** The gap between coarse and fine dual grids must be explicitly handled via stitching to ensure topological continuity [S1].

## Subtopics

### [Dual Grid Isosurface Extraction with Stitch Cells](10500-dual-grid-isosurface-extraction-with-stitch-cells.md)
This method extracts continuous, crack-free isosurfaces from cell-centered AMR data by constructing a dual grid that connects cell centers, thereby eliminating dangling nodes inherent in standard Marching Cubes on vertex-centered resampled data. It differs from previous crack-fixing solutions that create polygons, collapse polylines, or use triangle fans by using a dual grid and tessellated "stitch cells" to maintain topological consistency without interpolation artifacts.

## Operating characteristics
Not given in the sources.
## Background
Originated in the context of extracting isosurfaces from Adaptive Mesh Refinement (AMR) data, where standard resampling to vertex-centered formats introduces dangling nodes that cause cracks in Marching Cubes outputs [S1]. The method was developed to solve these discontinuities by leveraging the natural cell-center structure of AMR data [S1].
## Sources
[S1] Extraction of Crack-free Isosurfaces from Adaptive Mesh Refinement Data, https://doi.org/10.2312/vissym/vissym01/025-034, fetched 2026-10-26
