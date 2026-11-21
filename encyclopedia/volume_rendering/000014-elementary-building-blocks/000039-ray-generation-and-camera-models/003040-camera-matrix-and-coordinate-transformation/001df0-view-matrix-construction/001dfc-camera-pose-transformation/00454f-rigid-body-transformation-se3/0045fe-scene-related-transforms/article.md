# Scene-Related Transforms
## Overview
Scene-related transforms are a class of imaging transforms that operate primarily on scenes, as opposed to structure-related transforms which operate on objects extracted from scenes [S1]. They facilitate the transformation of information between scene, object, and view spaces, assuming the orientation and position relative to the scene or object coordinate system are known [S1]. This article builds on [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md) [G].
## Shared principles
The family applies the rigid transformations defined in [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md) to scene data [G]. It operates on both grey-level and binary scenes [S1].
## Subtopics

### Camera Coordinate Homogenization and Rigid Transformation
This sibling transform handles the specific homogenization and rigid transformation steps required for camera coordinates [G]. It is chosen when converting between camera-specific coordinate systems and the global scene space [G]. See [Camera Coordinate Homogenization and Rigid Transformation](4005-camera-coordinate-homogenization-and-rigid-transformation.md).

## Background
The classification of imaging transforms into scene-related and structure-related categories originates from the work of Udupa and Goncalves [S1]. These transforms are applicable to both grey-level and binary scenes, although their applicability to binary scenes is sometimes trivial [S1]. The transforms described are used to map information among scene, object, and view spaces [S1].
## Sources
[S1] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
