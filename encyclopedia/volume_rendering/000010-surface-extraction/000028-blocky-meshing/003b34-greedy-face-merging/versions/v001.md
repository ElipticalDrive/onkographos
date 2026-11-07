# Greedy face merging
Also called: Greedy meshing, Greedy face culling

## Overview
Greedy face merging is a family of surface extraction algorithms that reduce the vertex and primitive count of a voxel mesh by merging adjacent, coplanar faces of identical material into maximal axis-aligned quads [S1, S3, S4, unofficial]. It solves the problem of excessive geometry in naive voxel rendering (where every exposed voxel face becomes a separate quad) by scanning the volume along each of the three principal axes and greedily expanding rectangles of identical visible faces [S1, S3, S7, unofficial]. Unlike [Isosurface extraction](30-isosurface-extraction.md), which interpolates scalar fields to create smooth surfaces, greedy meshing preserves the exact blocky aesthetic of the discrete grid [S3, S13, unofficial]. It differs from [Blocky meshing](31-blocky-meshing.md) siblings like [Cuberille Method](1718-cuberille-method.md) by actively merging adjacent primitives into larger rectangles rather than just culling internal faces [S3, S4, unofficial].

## Shared principles
The core mechanism involves iterating over the three principal axes (normals) to process the voxel volume. For each normal direction $n \in \{0, 1, 2\}$ (representing X, Y, Z), the algorithm defines the tangent $t = (n + 1) \pmod 3$ and bitangent $b = (n + 2) \pmod 3$ [S1, S7, unofficial].

1.  **Sweep Slices:** The algorithm iterates through the volume along the normal axis $n$. For each slice index $s$ from $0$ to $D_n - 1$, it constructs a 2D mask $M$ of size $D_t \times D_b$.
2.  **Visibility Check:** For each cell $(t, b)$ in the slice at $s$, the algorithm determines if the face is exposed. A face is exposed if the voxel at $(x,y,z)$ (where the $n$-coordinate is $s$) has a different material or is solid while the neighbor in the normal direction is empty (or vice versa, depending on culling logic). Specifically, it compares $V(s, t, b)$ with $V(s - 1, t, b)$ if sweeping in the positive direction, or $V(s + 1, t, b)$ if sweeping in the negative direction [S7, S13, S19, unofficial]. The mask $M[t][b]$ is set to the material ID if exposed, or $0$ (or false) if not [S27, S31, unofficial].
3.  **Greedy Expansion:** The algorithm iterates through the mask $M$ in lexicographic order (row by row, then column by column) to find maximal rectangles [S22, S27, unofficial].
    *   It finds the first cell $(t_0, b_0)$ in $M$ that is non-zero (exposed) and not yet processed [S22, S27, unofficial].
    *   **Expand Width ($w$):** It increments $w$ starting at 1 while $t_0 + w < D_t$, the cell $(t_0 + w, b_0)$ is non-zero, and the material ID matches the starting cell's ID [S1, S27, unofficial].
    *   **Expand Height ($h$):** It increments $h$ starting at 1 while $b_0 + h < D_b$. For each potential height, it verifies that *all* cells in the rectangle defined by $[t_0, t_0 + w) \times [b_0, b_0 + h)$ have the same material ID and are non-zero [S1, S27, unofficial].
    *   **Emit Quad:** It creates a quad with origin $(x,y,z)$ corresponding to $(t_0, b_0, s)$, normal $n$, width $w$, and height $h$ [S27, unofficial].
    *   **Mark Processed:** It marks all cells in the rectangle $[t_0, t_0 + w) \times [b_0, b_0 + h)$ as processed (e.g., sets mask to 0 or updates a visited array) to prevent them from being part of another quad in this pass [S21, S27, unofficial].
    *   It advances the scan cursor to $t_0 + w$ [S27, unofficial].

**Data Structures:**
*   **Mask:** A 2D boolean or integer array of size $D_u \times D_v$ (where $u, v$ are the two axes perpendicular to the current sweep normal) used to track which cells in the current slice have been processed or to store the visibility/material state [S1, S7, S27, unofficial].
*   **Visited/Processed:** A 3D boolean array or bitmask array of size $D_x \times D_y \times D_z$ to ensure each voxel face is merged only once, preventing double-counting [S22, S28, unofficial].

**Parameters:**
*   **Merge Key:** By default, quads merge if they have the same material ID. Variants allow a custom `mergeKey(material, x, y, z)` function to include attributes like ambient occlusion, UV offset, or transparency [S12, S24, S30, unofficial].
*   **Boundary Policy:** Determines how out-of-bounds voxels are treated during visibility checks (e.g., always air, always solid) [S12, S13, unofficial].

## Subtopics

### [Greedy Meshing](1776-greedy-meshing.md)
Greedy meshing is a surface extraction algorithm that reduces the vertex and primitive count of a voxel mesh by merging adjacent, coplanar faces of identical material into maximal axis-aligned quads. It solves the problem of excessive geometry in naive voxel rendering (where every exposed voxel face becomes a separate quad) by scanning the volume along each of the three principal axes and greedily expanding rectangles of identical visible faces. Unlike [Isosurface extraction](30-isosurface-extraction.md), which interpolates scalar fields to create smooth surfaces, greedy meshing preserves the exact blocky aesthetic of the discrete grid. It differs from [Blocky meshing](31-blocky-meshing.md) siblings like [Cuberille Method](1718-cuberille-method.md) by actively merging adjacent primitives into larger rectangles rather than just culling internal faces.

### [3D Greedy Meshing](3519-3d-greedy-meshing.md)
3D Greedy Meshing extends 2D greedy meshing to three dimensions by extracting 2D slices from a 3D voxel grid along each axis and processing them with the 2D algorithm to generate a reduced set of faces representing the external surface. It solves the problem of high triangle counts in voxel-based rendering by merging adjacent visible faces of the same voxel type into larger quads, achieving a simplified mesh that has at most 8x the number of faces that a perfect mesh would have. It differs from its siblings by explicitly decomposing the 3D problem into six directional 2D sub-problems rather than operating directly on 3D volumes or using run-length encoding along axes.

### [Greedy Meshing for Opaque/Undefined Blocks](4917-greedy-meshing-for-opaque-undefined-blocks.md)
This method extends standard greedy meshing to handle voxel types that are either fully opaque or explicitly undefined (empty), ensuring that only visible faces of non-empty blocks are meshed while enforcing strict appearance definitions. It differs from general greedy meshing by introducing explicit assertions for missing block appearances and standardized UV/color padding for undefined blocks, preventing rendering artifacts in engines that require complete vertex attributes.

## Background
Not given in the sources.

## Sources
[S1] Greedy Meshing Algorithm, https://github.com/mauriciopoppe/greedy-meshing, 2023-10-01
[S3] Greedy Meshing, https://www.iquilezles.org/www/articles/greedy/greedy.htm, 2023-10-01
[S4] Greedy Meshing, https://www.youtube.com/watch?v=7M175LxJFjE, 2023-10-01
[S7] Greedy Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S12] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S13] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S19] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S20] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S21] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S22] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S24] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S27] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S28] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S30] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S31] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S6] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S9] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
[S16] Voxel Chunk Meshing, https://github.com/mauriciopoppe/greedy-meshing/blob/master/README.md, 2023-10-01
