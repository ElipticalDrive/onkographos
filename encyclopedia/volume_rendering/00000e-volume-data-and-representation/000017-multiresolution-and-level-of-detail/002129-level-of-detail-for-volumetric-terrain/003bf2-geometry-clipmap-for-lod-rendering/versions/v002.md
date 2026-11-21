# Geometry Clipmap for LOD Rendering
Also called: Volumetric terrain clipmap

## Overview
Geometry Clipmap for LOD Rendering is a level-of-detail technique for real-time volumetric terrain that replaces concentric 2D rings with 3D cubical shells to reduce vertex count for distant geometry [S1]. It differs from sibling methods like [Nested Clip-Boxes](13078-nested-clip-boxes.md) by operating on 2D heightmaps with regular grids, whereas Nested Clip-Boxes handles complex, rapidly changing 3D mesh topologies derived from arbitrary volumetric data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Viewer position, terrain function (implicit surface), clipmap shell parameters (thickness, LOD levels).
*   **Output:** Triangle meshes for each LOD shell, rendered with decreasing vertex density as distance from the viewer increases [S1].

**Data Structures:**
*   **Clipmap Shell List:** A list of world octant keys for every LOD shell. Each shell corresponds to a specific LOD level and spatial range around the viewer [S1].
*   **Queues:** Two separate queues manage tasks:
    1.  Volume modification tasks.
    2.  Contouring (mesh generation) tasks.
    *   Modifications take precedence over contouring [S1].
*   **Local CSG Queues:** Used to create data for an octant; this process is identical across all LOD grids [S1].

**Steps:**
1.  **Initialize Clipmap:** Create concentric cubical shells around the viewer. The innermost shell (LOD 0) has the highest resolution. Outer shells have progressively higher LOD levels (lower resolution) [S1].
2.  **Maintain Octant Keys:** For each shell, maintain a list of world octant keys representing the current active geometry patches [S1].
3.  **Update on Viewer Movement:**
    a.  Create a new list of keys for each shell based on the viewer's current position [S1].
    b.  Compare the new keys with the current list keys to identify:
        i.  Octants that have left the shell (need removal).
        ii. Octants that have entered the shell (need generation) [S1].
4.  **Task Management:**
    a.  Add volume modification tasks to the modification queue.
    b.  Add contouring tasks (mesh generation) to the contouring queue [S1].
    c.  Process queues with modification priority [S1].
5.  **Mesh Generation:**
    a.  For each active octant in a shell, use local CSG queues to generate mesh data [S1].
    b.  For LOD levels above zero, use world octants of the corresponding LOD grid to create meshes with lower effective resolution [S1].

**Formulas and Parameters:**
*   **LOD Resolution:** The resolution of a shell at LOD $L$ is lower than LOD $L-1$. The exact reduction factor is not specified, but the principle is that farther shells use higher LOD grids [S1].
*   **Exponential Data Requirement:** Accumulating data from lower LOD grids into a higher grid octant is not feasible. For example, creating a resampled data set in LOD 8 requires data from up to eight child octants from LOD 7. Each of those requires their own eight children, leading to $8^8 - 1 = 2097152$ octants needing processing [S1].

**Pitfalls:**
*   **Data Accumulation:** Do not attempt to accumulate existing data from lower LOD grids into an octant of a higher grid. This is not feasible because all required data from the lower grids must be fully available, leading to exponential processing requirements (e.g., 2097152 octants for LOD 8) [S1].

## Subtopics

### [Nested Clip-Boxes](13078-nested-clip-boxes.md)
A technique for massive procedural volumetric terrains that caches polygonal mesh geometry in nested cubic volumes. It handles complex 3D mesh topologies, unlike Geometry Clipmap which uses 2D heightmaps. Choose this for arbitrary volumetric data; choose Geometry Clipmap for regular grid heightmaps.

## Operating characteristics
*   **Time Complexity:** Not explicitly given. However, the method avoids exponential complexity by not accumulating data across LODs, processing each octant independently via local CSG queues [S1].
*   **Memory Complexity:** Not explicitly given. Depends on the number of active octants in the clipmap shells.
*   **Performance:** Improves GPU performance by reducing the number of vertices for distant meshes [S1].
*   **Strengths:** GPU-friendly, reduces vertex count for distant objects, straightforward update mechanism based on viewer movement [S1].
*   **Limitations:** Limited to 2D heightmaps with regular grids (unlike Nested Clip-Boxes which handles 3D volumetric data) [S1]. Accumulating data across LODs is computationally prohibitive [S1].
*   **When to use:** Real-time rendering of large volumetric terrain where the terrain can be represented as a 2D heightmap on a regular grid.
*   **When not to use:** When the terrain requires complex 3D volumetric topology or irregular grids.

## Background
Originated in the context of WebGL-based volumetric terrain rendering [S1]. The technique adapts conventional 2D geometry clipmaps (concentric rings) to 3D by using cubical shells composed of world octants [S1].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
