# Geometry Clipmap for LOD Rendering

Also called: Volumetric terrain clipmap

## Overview
Geometry Clipmap is a level-of-detail (LOD) technique for real-time rendering of volumetric terrain that reduces vertex count by rendering distant meshes at lower effective resolutions [S1]. It differs from conventional 2D clipmaps by using 3D cubical shells composed of world octants, allowing it to handle arbitrary volumetric models without preprocessing [S1].

## How it works
**Inputs:** Viewer position, viewer orientation, terrain resolution parameters, shell thickness.
**Outputs:** Triangle meshes for visible terrain shells.

**Data Structures:**
*   **LOD Grids:** A hierarchy of grids where LOD 0 is the finest resolution. Higher LOD indices correspond to coarser resolutions.
*   **World Octant Keys:** A unique identifier for each octant in the world coordinate system, used to track occupancy in clipmap shells.
*   **Clipmap Shells:** Concentric cubical shells surrounding the viewer. Each shell corresponds to a specific LOD level.
*   **Queues:**
    *   *Volume Modification Queue:* Prioritized queue for updating voxel data.
    *   *Contouring Queue:* Queue for generating meshes from voxel data.

**Steps:**
1.  **Initialize Clipmap Structure:** Create concentric cubical shells around the viewer. Each shell $i$ corresponds to LOD level $i$. The thickness of each shell is user-adjustable [S1].
2.  **Maintain Octant Lists:** For each shell, maintain a list of world octant keys currently occupying that shell [S1].
3.  **Update on Viewer Movement:**
    a.  Calculate the new set of world octant keys required for each shell based on the viewer's new position [S1].
    b.  Compare the new keys with the current list for each shell [S1].
    c.  Identify octants that have entered or left the shell [S1].
    d.  Add tasks for entering octants to the modification and contouring queues [S1].
    e.  Remove tasks for leaving octants [S1].
4.  **Process Queues:**
    a.  Execute volume modification tasks first, as they take precedence over contouring [S1].
    b.  Execute contouring tasks to generate meshes from the updated voxel data [S1].
5.  **Generate Meshes:**
    a.  For each octant in a shell at LOD $L > 0$, create a mesh with lower effective resolution than LOD 0 [S1].
    b.  Data for an octant is derived from local CSG (Constructive Solid Geometry) queues [S1].
    c.  The contouring process is identical across all LOD grids, relying on local data availability [S1].

**Formulas:**
*   **Octant Count for Resampling:** To resample data for a single octant at LOD $L$, if full data from lower grids is required, the number of octants to process is $8^L - 1$. For example, for LOD 8, this is $8^8 - 1 = 2097152$ octants [S1].

**Parameters:**
*   **Shell Thickness:** User-defined parameter controlling the depth of each LOD shell [S1].
*   **LOD Levels:** Discrete levels of resolution, with LOD 0 being the finest [S1].

**Edge Cases and Pitfalls:**
*   **Resampling Complexity:** Accumulating data from lower LOD grids into a higher grid octant is computationally infeasible because all required data from lower grids must be fully available. This leads to exponential processing requirements (e.g., $8^8 - 1$ octants for LOD 8) [S1].
*   **Queue Precedence:** Volume modification tasks must be processed before contouring tasks to ensure data consistency [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Contouring is $O(1)$ per octant if local CSG queues are used [S1]. Resampling from lower LODs is $O(8^L)$, which is infeasible for high LODs [S1].
*   **Memory Complexity:** Depends on the number of active octants in the clipmap shells.
*   **Strengths:** GPU-friendly, no preprocessing required, handles infinite procedural terrain, reduces vertex count for distant objects [S1].
*   **Limitations:** Exponential complexity for resampling from lower LODs prevents direct accumulation [S1].
*   **When to Use:** Real-time rendering of large-scale volumetric terrain where distant objects can be rendered at lower resolution [S1].
*   **When Not to Use:** When high-fidelity resampling from lower LODs is required for distant objects [S1].

## Background
The geometry clipmap approach was adapted for volumetric terrain rendering to improve GPU performance by reducing vertex counts for distant meshes [S1]. Conventional clipmaps use 2D concentric rings, while this method uses 3D cubical shells [S1]. The technique is supported by a world octree structure and allows for straightforward updates as the viewer moves [S1].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
