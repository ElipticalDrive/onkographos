# Depthless Cell Face Projection
## Overview
Depthless Cell Face Projection is a projection-based direct volume rendering method for curvilinear grids where each data value is mapped to color and opacity, and cell faces are drawn as Gouraud-shaded polygons without considering depth during compositing [S2, unofficial]. It sets itself apart from its parent [Hexahedron Projection for Curvilinear Grids](1111-hexahedron-projection-for-curvilinear-grids.md) by avoiding complex topological classification into triangle fans or strips, opting instead for direct polygonal projection prioritized for speed [S2, unofficial]. Unlike its sibling [Projection-Based Ray Casting for Curvilinear Grids](1756-projection-based-ray-casting-for-curvilinear-grids.md), it projects mesh geometry directly to the screen rather than casting rays through the volume [S2, unofficial].

## How it works
**Inputs:** A curvilinear grid mesh and a scalar field defined at each vertex [S2, unofficial].
**Outputs:** A 2D image with color and opacity values [S2, unofficial].

**Data Structures:**
Data structures record information for three adjoining faces of each cell to ensure each face is drawn only once [S2, unofficial]. The exact layout is not specified in the sources, but they must support efficient lookup to prevent duplicate rendering of shared faces between adjacent cells [S2, unofficial].

**Steps:**
1. Map each data value to a color and an opacity value [S2, unofficial].
2. Draw the faces of each curvilinear grid cell as Gouraud-shaded polygons, using the mapped values at the vertices [S2, unofficial].
3. Store information for three adjoining faces of the cell in data structures to ensure each face is drawn only once [S2, unofficial].

**Parameters:**
*   **Opacity:** Typically set to zero for maximum speed [S2, unofficial].
*   **Shading:** Gouraud shading is used [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Small Cells:** Small cells contribute the same intensity as large cells because depth is not considered [S2, unofficial]. On grids where cell size is inversely proportional to interest (finely gridded in areas of interest), this automatic weighting may be preferred by some scientists [S2, unofficial].
*   **Hardware Compositing Limitations:** Hardware compositing uses 8-bit intensity/opacity resolution [S1]. Small cells may contribute well under 1/256 of the maximum possible intensity and thus never appear in the image if depth were considered [S1][S2, unofficial]. The depthless method avoids ignoring data entirely but weights it improperly [S2, unofficial].
*   **Visual Artifacts:** Noticeable visual artifacts appear from some angles because the distance between cell faces is not taken into account [S2, unofficial]. These artifacts tend to delineate cell boundaries and probably would not be misinterpreted as data information [S1][S2, unofficial].

## Subtopics

### [Cell Face Projection with Depth](7453-cell-face-projection-with-depth.md)
A projection-based direct volume rendering method for curvilinear grids that addresses the lack of depth consideration in the depthless method. It maps sample locations to screen space and calculates depth, average color, and opacity for vertices, allowing opacity to be taken into consideration more accurately than its sibling [Depthless Cell Face Projection](6692-depthless-cell-face-projection.md). Choose this variant when physical depth weighting and accurate opacity accumulation are required over maximum rendering speed. [Cell Face Projection with Depth](7453-cell-face-projection-with-depth.md)

## Operating characteristics
**Time Complexity:** Not given in the sources [S1][S2, unofficial].
**Memory Complexity:** Not given in the sources [S1][S2, unofficial].
**Measured Performance:** Approximately one second for 40,000 cells on a uniprocessor R3000-based Silicon Graphics VGX [S1][S2, unofficial].
**Strengths:** Extremely fast and trivial to implement [S2, unofficial]. It prevents small cells from being ignored due to hardware compositing resolution limits [S2, unofficial].
**Limitations:** Lacks physical depth weighting, leading to improper weighting of cell contributions [S2, unofficial]. Produces visual artifacts delineating cell boundaries from certain angles [S1][S2, unofficial].
**When to Use:** When speed is the primary concern and the grid has many small cells that might be lost in hardware compositing [S2, unofficial]. When the automatic weighting of small cells (often high-interest areas) is desirable [S2, unofficial].
**When Not to Use:** When physical accuracy of depth and opacity accumulation is required [S2, unofficial]. When visual artifacts from cell boundaries are unacceptable [S2, unofficial].

## Background
The method was described in the 1993 paper "Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering" [S1]. It was developed as a simple, fast alternative to more complex volume rendering techniques for exploring curvilinear grids [S2, unofficial]. The method is part of a suite of techniques designed to handle the specific challenges of curvilinear meshes, such as non-planar faces and varying cell sizes [S2, unofficial].

## Sources
[S1] Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering, https://ntrs.nasa.gov/api/citations/19930015755/downloads/19930015755.pdf, fetched 2026-10-24
[S2] NASA Technical Reports Server (NTRS) 19930015755: Rapid exploration of curvilinear grids using direct volume rendering, https://archive.org/details/NASA_NTRS_Archive_19930015755, fetched 2026-10-26 (unofficial)
