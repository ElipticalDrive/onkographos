# Tile-based Depth Sorting

Also called: STP

## Overview
Tile-based Depth Sorting is a rasterization strategy designed to mitigate "popping" artifacts in 3D Gaussian Splatting by replacing per-ray depth sorting with tile-based sorting [S1][S2, unofficial]. It ensures a consistent depth ordering within a local region, addressing the sensitivity of alpha-blending order to viewpoint rotation [S1][S2, unofficial].

## How it works
Inputs: A list of projected 2D Gaussian splats. Outputs: A tile-sorted list of splats ready for rasterization.
Data structures: Not given in the sources.
Numbered steps:
1. Replace the standard per-ray depth sorting mechanism with a tile-based sorting approach [S1][S2, unofficial].
2. Ensure that the depth order of Gaussians remains consistent across neighboring pixels within a tile, preventing visual artifacts caused by inconsistent sorting when the viewpoint changes [S1][S2, unofficial].
Formulas: Not given in the sources.
Parameters: Not given in the sources.
Edge cases and pitfalls: Not given in the sources.
Pseudocode: Not given in the sources.

## Subtopics

### [Per-tile Sorting and Per-pixel Resorting](9984-per-tile-sorting-and-per-pixel-resorting.md)
Per-tile sorting and per-pixel resorting is a rasterization strategy designed to mitigate "popping" artifacts in surface-based volumetric representations (such as 2DGS and QGS) where Gaussian distributions are concentrated on surfaces. It addresses the sensitivity of alpha-blending order by replacing inconsistent per-ray depth sorting with a two-stage approach: coarse global ordering via per-tile depth approximation and fine local ordering via per-pixel resorting. This method sets itself apart from standard 3D Gaussian Splatting by enforcing strict consistency between forward rendering and backward gradient computation traversal orders, thereby eliminating streak-like and blocky artifacts introduced by single-ray approximations.

## Operating characteristics
Not given in the sources.

## Background
StopThePop [S1][S2, unofficial] points out that 3DGS tends to cheat view-dependent effects by popping 3D Gaussians due to the per-ray depth sorting, which leads to less faithful results when the viewpoint is rotated [S1][S2, unofficial]. To mitigate the potential of popping 3D Gaussians, StopThePop [S1][S2, unofficial] replaces the per-ray depth sorting with tile-based sorting to ensure consistent sorting order in a local region [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
