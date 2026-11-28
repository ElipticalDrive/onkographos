# Points representation
Also called: Point-based volume rendering
## Overview
Points representation renders a dataset as a collection of points by extracting its surface mesh [S1, S2]. It is distinguished from [Surface](18767-surface.md) by rendering only discrete vertices rather than filled polygons, and from [Wireframe](18768-wireframe.md) by omitting edge connectivity information [G].
## How it works
**Inputs and Outputs:**
- Input: An input dataset containing geometric cells and associated point coordinates [S1, S2].
- Output: A rendered image composed of point markers at the extracted surface vertices [S1, S2].
**Data Structures:**
The method relies on an extracted surface mesh [S1, S2]. The specific internal layout of the mesh is not detailed in the sources [S1, S2].
**Steps:**
1. Extract the surface mesh from the input dataset [S1, S2].
2. Render the extracted mesh as a collection of points [S1, S2].
3. Apply coloring using either a single solid color for all points, or scalar coloring (pseudocoloring) by selecting a data array to map to colors for each point [S1, S2].
**Formulas:**
Not given in the sources.
**Parameters:**
Not given in the sources.
**Edge Cases and Pitfalls:**
- The method depends on the successful extraction of a surface mesh [S1, S2].
- As a point-based visualization, it does not convey surface continuity or curvature as effectively as solid surface rendering [G].
## Operating characteristics
Time and memory complexity: Not given in the sources [S1, S2].
Measured performance: Not given in the sources [S1, S2].
Scaling: Not given in the sources [S1, S2].
Strengths and limitations: Not given in the sources [S1, S2].
When to use it and when not: Not given in the sources [S1, S2].
## Background
**Subtopics:**
- [Surface](18767-surface.md): Renders the extracted mesh as solid polygons. It sets itself apart by conveying surface continuity and shading. Choose it when surface continuity and shading are needed [G].
- [Wireframe](18768-wireframe.md): Renders the extracted mesh as lines connecting vertices. It sets itself apart by preserving cell boundary connectivity. Choose it when cell boundaries and connectivity are important [G].
- [Feature Edges](18770-feature-edges.md): Highlights prominent edges (sharp angles or boundaries) from the mesh. It sets itself apart by isolating geometric features. Choose it when specific geometric features need emphasis [G].
**Background:**
The Points representation is a standard visualization mode in ParaView [S1, S2]. It belongs to a family of surface-based representations that rely on mesh extraction [S1, S2].
## Sources
[S1] 4. Displaying data — ParaView Documentation 5.8.1 documentation, https://docs.paraview.org/en/v5.9.1/UsersGuide/displayingData.html, fetched 2026-10-29
[S2] 5. Displaying data — ParaView Documentation 6.2.0 documentation, https://docs.paraview.org/en/latest/UsersGuide/displayingData.html, fetched 2026-10-29
