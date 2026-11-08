# Dual Grid Isosurface Extraction with Stitch Cells

Also called: Dual Grid Method, First Approach

## Overview
This method extracts continuous, crack-free isosurfaces from cell-centered Adaptive Mesh Refinement (AMR) data by constructing a dual grid that connects cell centers, thereby eliminating dangling nodes inherent in standard Marching Cubes on vertex-centered resampled data [S1, unofficial]. It differs from previous crack-fixing solutions that create polygons, collapse polylines, or use triangle fans by using a dual grid and tessellated "stitch cells" to maintain topological consistency without interpolation artifacts [S1, unofficial].

## How it works
**Inputs:**
*   AMR hierarchy of cell-centered scalar data.
*   Isosurface value $v$.

**Outputs:**
*   A manifold triangle mesh representing the isosurface.

**Data Structures:**
*   **Dual Grid:** A grid where vertices correspond to the centers of the original AMR cells. Edges connect centers of adjacent cells.
*   **Stitch Cells:** Tessellated geometry filling gaps between hierarchy levels. In 3D, these must share only vertices, edges, and complete quadrilateral faces [S1, unofficial]. Tetrahedra are not used because cells must share quadrilaterals as faces [S1, unofficial].

**Steps:**

1.  **Construct Dual Grid:**
    Connect the centers of adjacent cells in the AMR hierarchy to form the dual grid vertices and edges [S1, unofficial].

2.  **Apply Marching Cubes:**
    Perform Marching Cubes on the dual grid using the original cell-centered values associated with the dual vertices. Linear interpolation along edges is permitted as it is inherent to Marching Cubes, but no other interpolation is performed [S1, unofficial].

3.  **Identify Gaps:**
    Locate regions between hierarchy levels where fine and coarse patches meet. These gaps contain dual grid vertices/edges that are not part of the standard dual grid connectivity of the coarse or fine patches [S1, unofficial].

4.  **Tessellate Gaps with Stitch Cells:**
    Fill the identified gap regions using stitch cells. The tessellation must satisfy:
    *   Only the gap region is tessellated.
    *   The complete gap region is tessellated.
    *   Stitch cells share only vertices, edges, and complete faces (quadrilaterals in 3D) with the surrounding geometry [S1, unofficial].

5.  **Handle Multiple Patches:**
    If multiple fine patches are adjacent, merge them by "upgrading" edges to quadrilaterals and vertices to edges to handle the connectivity correctly [S1, unofficial]. Specific cases include:
    *   All coarse grid points unrefined.
    *   Two neighboring coarse grid points refined.
    *   Two diagonally opposed coarse grid points refined.
    *   All coarse grid points refined [S1, unofficial].

**Pitfalls:**
*   Standard Marching Cubes on cell-centered data creates dangling nodes (vertices present only in fine levels), causing cracks [S1, unofficial].
*   Resampling to vertex-centered data requires consistent values to avoid problems; linear interpolation avoids problems but introduces artifacts [S1, unofficial].
*   In 3D, stitch cells cannot use tetrahedra because cells must share quadrilaterals as faces [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Tested on an AMR simulation of star cluster formation (Root level 32x32x32, Data set: Greg Bryan, Theoretical Astronomy Group, MIT) [S1, unofficial].
*   **Strengths:** Uses original data values for Marching Cubes; no dangling nodes; avoids interpolation artifacts [S1, unofficial].
*   **Limitations:** Requires complex tessellation logic for gaps and multiple patch adjacency [S1, unofficial].

## Background
This method is described in the context of "Current State of the Art in Adaptive Mesh Refinement Visualization" [S1, unofficial]. It is contrasted with previous crack-fixing solutions mostly in the context of Octree-based hierarchies, such as creating polygons to fit cracks [Shu et al., 1995], collapsing polylines to lines [Shekhar et al., 1996], and creating triangle fans [Westermann et al., 1999] [S1, unofficial]. The dual grid approach is labeled the "First Approach" in this context [S1, unofficial].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
