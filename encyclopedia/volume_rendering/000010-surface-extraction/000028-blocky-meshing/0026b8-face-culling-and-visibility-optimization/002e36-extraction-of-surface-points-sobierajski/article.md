# Extraction of Surface Points (Sobierajski)
Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It sets itself apart from sibling methods like [Face Culling](8735-face-culling.md) by operating in object space to generate a pre-filtered list of surface points rather than calculating face visibility per voxel during rasterization. This reduces the rendering load by eliminating occluded and back-facing voxels before they reach the image plane [S1, unofficial].

## How it works
**Inputs:** Volume data grid; set of viewing directions (typically six orthographic views along coordinate axes) [S1, unofficial].
**Outputs:** A list of unique boundary voxels with attributes (coordinates, normal vectors) [S1, unofficial].

**Data Structures:**
*   `BoundaryVoxelList`: A dynamic list or hash set storing extracted boundary voxels. Each entry contains:
    *   `position`: 3D integer coordinates $(x, y, z)$ of the voxel.
    *   `normal`: 3D float vector representing the surface normal at the voxel boundary.
    *   `density`: Original density value (for cutting plane support) [S1, unofficial].

**Steps:**
1.  **Visibility Determination:** For each of the specified viewing directions, determine which boundary voxels are potentially visible. For the standard six orthographic views, this is performed efficiently using a 2D boundary tracing algorithm on slices perpendicular to each coordinate axis [S1, unofficial].
2.  **Extraction and Deduplication:** Extract the boundary voxels identified as visible. Store them in `BoundaryVoxelList`, ensuring duplicate elements (voxels visible from multiple directions) are removed [S1, unofficial].
3.  **Attribute Assignment:** For each voxel in the list, compute and store necessary rendering attributes, specifically coordinates and normal vectors [S1, unofficial].
4.  **Back-Face Culling (Voxel Culling):** To further reduce the list size for a specific viewing direction, apply back-face culling logic [S1, unofficial].
    *   Let $\mathbf{n}$ be the surface normal of the voxel boundary.
    *   Let $\mathbf{v}$ be the viewing direction vector.
    *   If $\mathbf{n} \cdot \mathbf{v} > 0$, the voxel is a back face and is removed from the list for that view [S1, unofficial].
5.  **Projection:** Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto several pixels depending on the viewing direction [S1, unofficial].
6.  **Rendering:**
    *   Shade the projected boundary surface points according to lighting conditions [S1, unofficial].
    *   If cutting planes are used, render voxels intersected by the cutting planes by projecting their original density values onto the image plane [S1, unofficial].

**Parameters:**
*   **Viewing Directions:** Typically six orthographic views (along $\pm x, \pm y, \pm z$). Increasing the number of directions improves the approximation of the visible surface set [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** Adjacent voxels may map to non-adjacent pixels, creating visual holes. Mitigated by projecting each voxel onto multiple pixels [S1, unofficial].
*   **Precision:** Dot product calculation for back-face culling must handle floating-point precision carefully to avoid incorrectly culling grazing angles [G].

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It differs from the standard method by sacrificing completeness for speed, extracting only a subset of boundary voxels uniformly [S1, unofficial]. The extracted surface samples are converted to geometrical primitives (e.g., crosslines perpendicular to the surface normal) and projected onto the image plane [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the 2D boundary tracing on slices and the projection step. The number of voxels to be projected is reduced by back-face culling, making it faster than rendering the full boundary set [S1, unofficial].
*   **Memory Complexity:** Requires storage for the `BoundaryVoxelList`. The size is proportional to the surface area of the object, significantly less than the full volume [S1, unofficial].
*   **Strengths:** Fast direct rendering; supports flexible manipulation of extracted voxels (e.g., cutting planes); efficient visibility calculation using orthographic views [S1, unofficial].
*   **Limitations:** Approximate visibility (six views may miss some surfaces); potential for holes if not projected onto multiple pixels [S1, unofficial].
*   **When to use:** Medical imaging systems requiring fast previewing and cutting operations [S1, unofficial].

## Background
Proposed by Sobierajski [116] as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical imaging systems where cutting operations are important [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
