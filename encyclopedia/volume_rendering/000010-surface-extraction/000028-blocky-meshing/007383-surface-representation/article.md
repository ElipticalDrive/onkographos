# Surface representation

Also called: Surface With Edges Representation

## Overview
Surface representation relies on extracting a surface mesh from a discretized dataset and rendering it as a solid surface [S1, S2]. It is distinguished from volume rendering by its reliance on explicit mesh extraction rather than direct voxel traversal, and from outline or points representations by its solid geometric fill [S1, S2].

## How it works
**Inputs and outputs:**
- Input: A discretized dataset containing scalar values at vertices or cell centers [S1, S2].
- Output: A rendered solid surface mesh, optionally with cell boundaries highlighted [S1, S2].

**Data structures:**
The method requires an intermediate surface mesh data structure [S1, S2]. The mesh consists of vertices and connectivity lists defining the surface geometry [G].

**Steps:**
1. **Extract Surface Mesh:** Generate a mesh representing the isosurface or boundary of the dataset using a surface extraction algorithm [S1, S2].
2. **Configure Rendering Properties:**
   - Set the representation type to "Surface" or "Surface With Edges" [S1, S2].
   - **Coloring:** Choose between a single solid color or a data array for scalar coloring (pseudocoloring) [S1, S2].
   - **Opacity:** Set the opacity value. A value of 0 implies total transparency (invisibility), while 1 implies total opacity [S1, S2].
3. **Render:**
   - For **Surface**: Render the extracted mesh as a solid surface [S1, S2].
   - For **Surface With Edges**: Render the solid surface and additionally highlight the cell boundaries (edges of the mesh cells) [S1, S2].

**Parameters:**
- **Opacity:** Float in $[0, 1]$. 0 is invisible, 1 is opaque [S1, S2].
- **Solid Color:** Color value picked via the UI's Coloring combo-box [S1, S2].
- **Scalar Array:** Name of the data array to use for pseudocoloring [S1, S2].

**Edge cases and pitfalls:**
- Translucent geometries (opacity < 1) should be avoided during interactive exploration to maintain performance, but are suitable for final image generation [S1, S2].
- Scalar coloring options have no effect on Outline representation, but are valid for Surface and Surface With Edges [S1, S2].

## Operating characteristics
- **Performance:** Slower than Outline representation because it requires mesh extraction and rendering of filled polygons rather than just a bounding box [S1, S2].
- **Memory:** Requires memory to store the extracted surface mesh in addition to the original dataset [G].
- **Strengths:** Provides a clear, solid view of the data's shape and internal structure (via coloring) [S1, S2].
- **Limitations:** Not given in the sources.

## Background
Surface representation is a standard visualization technique in scientific visualization software, prominently featured in ParaView [S1, S2]. It is part of a family of display representations that includes Outline, Points, Wireframe, and Volume [S1, S2].

## Sources
[S1] 4. Displaying data — ParaView Documentation 5.8.1 documentation, https://docs.paraview.org/en/v5.9.1/UsersGuide/displayingData.html, fetched 2026-10-29
[S2] 5. Displaying data — ParaView Documentation 6.2.0 documentation, https://docs.paraview.org/en/latest/UsersGuide/displayingData.html, fetched 2026-10-29
