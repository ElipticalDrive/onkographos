# Multi-view projection
Also called: Multi-view rasterization, Axis-aligned voxelization, Orthographic projection voxelization

## Overview
Multi-view projection is a family of rasterization-based voxelization techniques that convert triangle meshes into voxel grids by rendering scene geometry from multiple distinct camera viewpoints. The core problem it solves is ensuring topological correctness (specifically 6-separation or tunnel-freedom) and complete coverage of the volume, which single-view methods often fail to achieve due to perspective distortion or degenerate alignments. It sets itself apart from single-view methods by requiring multiple passes (typically 2, 3, or 6) to guarantee that every voxel intersected by a triangle is covered by at least one view, provided the views are sufficiently separated [Parent].

## Shared principles
All methods in this family share the following mechanism and mathematical constraints:

1.  **Multiple Viewpoints:** The scene is rasterized from at least two distinct camera positions. The views are typically aligned with the principal axes of the scene's bounding box (X, Y, Z) to ensure orthogonality and simplify coordinate mapping.
2.  **Coverage Guarantee:** The set of view directions is chosen such that for any triangle in the scene, at least one view direction is not parallel to the triangle's plane. This prevents the triangle from projecting to a line segment or point, which would result in missing voxels.
3.  **Voxel Space Mapping:** Fragments generated during rasterization are mapped directly to voxel indices. This is typically achieved using a projection matrix that scales clip-space coordinates to the voxel grid resolution and offsets them by the bounding box minimum.
4.  **Atomic Writes:** When multiple triangles or multiple views write to the same voxel location, atomic operations are required to prevent race conditions in parallel implementations [G].
5.  **6-Separation:** The resulting voxel grid aims to be 6-separating (tunnel-free), meaning that any path through the volume must pass through occupied voxels, preventing diagonal tunneling artifacts. This is ensured by the multi-view coverage and the specific rasterization rules (e.g., edge-crossing tests in single-view variants, or redundant coverage in multi-view variants).

## Subtopics

### [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md)
Uses a single 2D rasterization pass along the dominant axis of the triangle's normal. It relies on edge-crossing tests for boundary voxels to ensure 6-separation. Choose this for simple, convex shapes where performance is critical and the dominant axis is well-defined.

### [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md)
Rasterizes geometry into a 3D texture using three orthographic cameras aligned with the scene's bounding box axes. It ensures coverage by using multiple views. Choose this for general-purpose voxelization where robustness against degenerate triangles is required.

### [3-Pass Voxelization](5300-3-pass-voxelization.md)
A specific case of multi-view voxelization using exactly three orthogonal passes (X, Y, Z). It eliminates cracks and missing data by guaranteeing that every intersected voxel is covered by at least one view. Choose this for high-fidelity voxelization of complex meshes where crack-free surfaces are essential.

### [Volumetric Representation via Depth Map Registration](6435-volumetric-representation-via-depth-map-registration.md)
Constructs a 3D volume from multi-view images by registering estimated depth maps into a voxel grid. It addresses non-Lambertian reflections by using individual minimum cost per view. Choose this for scenes with complex lighting or specularities where standard rasterization might fail.

## Background
The concept of multi-view projection for voxelization originates from the need to create robust, tunnel-free voxel representations for volume rendering and real-time global illumination. Early methods relied on single-view rasterization, which suffered from geometric distortion and gaps. The introduction of multi-view approaches, particularly those using axis-aligned orthographic projections, provided a solution by ensuring that every voxel intersected by a triangle is covered by at least one view [S1, unofficial][S2, unofficial]. This family of methods has become standard in GPU-based voxelization pipelines due to their simplicity and compatibility with hardware rasterizers.

## Sources
[S1] Unofficial notes on Multi-view projection and its variants, accessed 2024.
[S2] Unofficial notes on 3-Pass Voxelization, accessed 2024.
