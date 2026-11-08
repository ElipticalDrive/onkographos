# Irregular Data Ray Traversal
Also called: Software Ray Casting for Irregular Grids
## Overview
Irregular Data Ray Traversal renders curvilinear or unstructured data (such as tetrahedral or hexahedral meshes) by determining an initial cell and iteratively traversing subsequent cells along a ray [S1]. It differs from regular grid traversal by relying on explicit mesh connectivity and acceleration structures over boundary faces rather than implicit coordinate mapping [S1].
## Shared principles
The method consists of two primary phases: initial cell location and iterative traversal [S1].

1.  **Initial Cell Location**: The entry point of the ray into the volume is determined by intersecting the ray with an acceleration structure built over the volume's boundary faces [S1]. Boundary faces are identified as those not shared between two cells [S1]. Standard ray-triangle or ray-face intersection techniques are used to find the closest intersection [S1].
2.  **Iterative Traversal**: Once the initial cell is identified, the ray is traversed through the mesh [S1]. The algorithm moves from the current cell to the next adjacent cell along the ray direction [S1]. This process repeats until the ray exits the volume or the last cell is reached [S1].

**Adjacency Requirement**: Incremental traversers require explicit adjacency information [S1]. Consequently, complete faces must be shared between adjacent cells [S1]. This constraint prohibits sliding interfaces or non-conforming meshes where faces do not perfectly align [S1].

## Subtopics
This family does not have distinct child algorithms listed in the provided tree structure beyond the parent Garrity Ray Casting, which is a specific implementation context. The general technique applies to both tetrahedral and hexahedral meshes [S1].

## Background
The rendering of irregular data sets using software ray casting was proposed as early as 1990 [S1]. Early works include [Gar90, Lev90, WCA* 90] [S1]. Subsequent developments involved various acceleration structures, such as projecting boundary faces onto the image plane [S1]. Researchers such as [BKS97, HK98, HK99, KN91] contributed to methods for locating the initial cell [S1]. The approach is detailed in "Interactive Volume Rendering with Ray Tracing" [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
