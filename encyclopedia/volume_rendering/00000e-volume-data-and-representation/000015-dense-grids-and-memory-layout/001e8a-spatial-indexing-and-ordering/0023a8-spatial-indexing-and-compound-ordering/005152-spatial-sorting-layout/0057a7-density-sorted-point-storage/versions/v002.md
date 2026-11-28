# Density-sorted point storage

Also called: Density sorting

## Overview
Density-sorted point storage is a spatial indexing and ordering method for point-based volume rendering that organizes points strictly by their scalar density values rather than their spatial proximity [S1, book]. Unlike spatially coherent layouts, this method sacrifices neighborhood information to enable rapid culling of transparent or irrelevant points during the rendering pipeline [S1, book]. It is distinct from slice-sorted variants by lacking implicit depth ordering, requiring explicit depth sorting or binning during rendering [S1, book]. It builds on [Spatial Sorting Layout](14435-spatial-sorting-layout.md).

## How it works
**Inputs and Outputs:**
- **Input:** A set of points, each defined by spatial coordinates $(x, y, z)$, a scalar density value, and optionally gradient vectors [S1, book].
- **Output:** A list of points ordered by density [S1, book].

**Data Structures:**
- **Point Structure:** Each point stores $(x, y, z)$ coordinates and the density value. If gradients are used, they are also stored [S1, book]. No spatial adjacency metadata is maintained [S1, book].
- **Storage Layout:** A flat array or list of point structures, sorted by density [S1, book].

**Steps:**
1. **Transform:** View-transform all points before culling to determine visibility relative to the camera or viewing pyramid [S1, book].
2. **Cull:** Iterate through the sorted points and discard those assigned transparent values in the transfer function [S1, book].
3. **Depth Sort:** Depth-sort the remaining visible points during rendering, or toss them into depth bins [S1, book].

**Pitfalls:**
- **Loss of Neighborhood:** Neighborhood relations are completely lost, preventing optimizations that rely on spatial coherence [S1, book].
- **View Transformation Overhead:** All points must be transformed to view space before culling, which adds computational cost [S1, book].
- **Depth Sorting Cost:** The lack of spatial coherence necessitates depth sorting or binning during rendering [S1, book].

## Subtopics

### Slice-sorted point storage
A compromise method where points are sorted by density within individual volume slices rather than globally [S1, book]. This provides implicit depth-ordering when used with axis-aligned sheet-buffer methods, reducing the need for explicit depth sorting compared to global density sorting [S1, book]. Choose this when a balance between culling efficiency and depth-ordering simplicity is needed for axis-aligned rendering [S1, book].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Efficient culling of transparent points; simple data structure [S1, book].
- **Limitations:** No spatial coherence; requires view transformation for culling; requires depth sorting/binning [S1, book].
- **When to Use:** Scenes with large transparent regions where culling efficiency outweighs the cost of depth sorting [S1, book].
- **When Not to Use:** Scenes requiring high spatial coherence or where depth sorting is prohibitively expensive [S1, book].

## Background
Density sorting was identified in point-based volume visualization as the scheme providing the least spatial coherence among point storage methods [S1, book]. The source contrasts it with RLE-based approaches, which do not allow easy density-based culling, and octree decompositions, which provide depth sorting only at the node level [S1, book].

## Sources
[S1] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
