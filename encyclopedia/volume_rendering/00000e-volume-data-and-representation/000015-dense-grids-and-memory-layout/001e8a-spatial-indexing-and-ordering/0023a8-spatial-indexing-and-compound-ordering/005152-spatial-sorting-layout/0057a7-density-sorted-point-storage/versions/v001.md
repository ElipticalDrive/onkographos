# Density-sorted point storage

Also called: Density sorting

## Overview
Density-sorted point storage is a spatial indexing and ordering method for point-based volume rendering that organizes points strictly by their scalar density values rather than their spatial proximity. Unlike spatially coherent layouts, this method sacrifices neighborhood information to enable rapid culling of transparent or irrelevant points during the rendering pipeline. It is distinct from slice-sorted variants by lacking implicit depth ordering, requiring explicit depth sorting or binning during rendering.

## How it works
**Inputs and Outputs:**
- **Input:** A set of $N$ points, each defined by spatial coordinates $(x, y, z)$, a scalar density value $\rho$, and optionally gradient vectors $\nabla \rho$.
- **Output:** A sorted list of points ordered by ascending or descending density.

**Data Structures:**
- **Point Structure:** Each point stores $(x, y, z)$ coordinates and $\rho$. If gradients are used, $\nabla \rho$ is also stored. No spatial adjacency metadata is maintained.
- **Storage Layout:** A flat array or list of point structures, sorted by $\rho$.

**Steps:**
1. **Transform:** Transform all point coordinates from object space to view space. This is required before culling to determine visibility relative to the camera [S1, book].
2. **Cull:** Iterate through the sorted points. Points with density values mapped to transparent regions in the transfer function are discarded immediately. This leverages the density sort to skip irrelevant data efficiently [S1, book].
3. **Depth Sort:** The remaining visible points must be depth-sorted to handle occlusion correctly. This can be done by:
   - Explicitly sorting the visible points by their view-space $z$-coordinate.
   - Tossing points into depth bins (e.g., z-buffer bins) to approximate depth order [S1, book].
4. **Render:** Render the depth-sorted points using a splatting or ray-marching approach.

**Pitfalls:**
- **Loss of Neighborhood:** Neighborhood relations are completely lost, preventing optimizations that rely on spatial coherence or local gradient estimation without storing extra data [S1, book].
- **View Transformation Overhead:** All points must be transformed to view space before culling, which can be costly if the view changes frequently [S1, book].
- **Depth Sorting Cost:** The lack of spatial coherence necessitates depth sorting or binning during rendering, increasing computational overhead compared to back-to-front sorted spatial layouts [S1, book].

## Subtopics

### Slice-sorted point storage
A compromise method where points are sorted by density within individual volume slices rather than globally [S1, book]. This provides implicit depth ordering when used with axis-aligned sheet-buffer methods, reducing the need for explicit depth sorting compared to global density sorting [S1, book]. Choose this when a balance between culling efficiency and depth-ordering simplicity is needed for axis-aligned rendering.

## Operating characteristics
- **Time Complexity:** Sorting by density is $O(N \log N)$. Culling is $O(N)$ but efficient due to sorted order. Depth sorting adds $O(K \log K)$ or $O(K)$ with binning, where $K$ is the number of visible points.
- **Memory Complexity:** $O(N)$ for storing points. No additional spatial index structures are required, but coordinates and gradients must be stored explicitly since neighborhood info is lost [S1, book].
- **Strengths:** Efficient culling of transparent points; simple data structure.
- **Limitations:** No spatial coherence; requires view transformation for culling; requires depth sorting/binning.
- **When to Use:** Scenes with large transparent regions where culling efficiency outweighs the cost of depth sorting.
- **When Not to Use:** Scenes requiring high spatial coherence or where depth sorting is prohibitively expensive.

## Background
Originated in the context of point-based volume visualization, density sorting was identified as a scheme that provides the least spatial coherence among point storage methods [S1, book]. It was discussed in contrast to RLE-based and octree-based approaches, highlighting its trade-off between culling efficiency and the loss of spatial relationships [S1, book].

## Sources
[S1] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
