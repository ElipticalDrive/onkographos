# Hierarchical DDA Traversal
Also called: Hierarchical Digital Differential Analyzer, Multi-level DDA, VolumeHDDA

## Overview
Hierarchical DDA Traversal extends standard DDA [DDA](2323-dda.md) to efficiently traverse sparse, multi-resolution volume structures by adapting step sizes to the current tree level. Unlike sibling methods that step voxel-by-voxel [Amanatides-Woo Voxel Traversal](3132-amanatides-woo-voxel-traversal.md) or use fixed-resolution grids [3D Bresenham Voxel Traversal](3951-3d-bresenham-voxel-traversal.md), this family skips empty internal nodes or adjusts mip-levels dynamically to balance speed and precision.

## Shared principles
The core mechanism involves maintaining DDA state (`mNext`, `mDelta`, `mStep`) relative to a current dimension `mDim`. When the ray exits the current spatial unit (node or brick), the state is re-initialized for the new resolution `dim_new` using clamping to prevent out-of-bounds access [G].

## Subtopics

### [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md)
Uses multi-resolution leapfrogging on VDB structures. Chooses when explicit node skipping is required.

### [VolumeHDDA](2353-volumehdda.md)
Recursive traversal templated on tree type. Chooses when discrete tile/voxel intersection detection with time-span management is needed.

### [Multi-level DDA with Adaptive Mip Selection](4086-multi-level-dda-with-adaptive-mip-selection.md)
Combines DDA with stochastic sampling and adaptive mip selection. Chooses for GPU-friendly, branchless traversal of dense volumes with majorant textures.

## Background
Originated in sparse voxel grid rendering, specifically for VDB structures . Developed to address the inefficiency of voxel-by-voxel traversal in large, sparse datasets.

## Sources
[S1] OpenVDB Documentation, https://www.openvdb.org, 2023-10-01
[S5] OpenVDB Source Code (hdda.h), https://github.com/AcademySoftwareFoundation/openvdb, 2023-10-01
