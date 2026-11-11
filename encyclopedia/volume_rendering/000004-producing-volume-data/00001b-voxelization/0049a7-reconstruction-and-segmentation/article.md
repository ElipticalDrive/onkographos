# Reconstruction and Segmentation

Also called: Volumetric Reconstruction, Volume Segmentation

## Overview
This family encompasses methods that generate structured 3D volumes from non-volumetric sources (such as point clouds, meshes, or image sets) for the purpose of rendering or analysis. It solves the problem of converting continuous or sparse representations into discrete voxel grids with semantic or photometric labels. It is distinguished from sibling methods like [Voxelization](12-voxelization.md) by focusing on the semantic assignment (segmentation) or photometric consistency (coloring) of the volume content rather than simple geometric occupancy.

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Voronoi-like Segmentation Volume Generation](4218-voronoi-like-segmentation-volume-generation.md)
- **What it is**: Generates a 3D volume where each voxel is assigned a label corresponding to the index of its nearest neighbor in a synthetic, Poisson-distributed point set.
- **Distinction**: It creates spatially coherent segmentation labels without relying on input geometry, unlike [Voxelization](12-voxelization.md) which relies on geometric primitives.
- **When to choose**: When structured, spatially coherent labels are needed for volume data without explicit geometric boundaries.
- **Link**: [Voronoi-like Segmentation Volume Generation](4218-voronoi-like-segmentation-volume-generation.md)

### [Voxel Coloring](4418-voxel-coloring.md)
- **What it is**: Computes photo-realistic 3D models from multiple color images by assigning colors (radiances) to voxels, guaranteeing consistency with input images.
- **Distinction**: Unlike [Voxel Carving](1213-voxel-carving.md) which starts with an opaque volume and removes inconsistent voxels, standard Voxel Coloring (in its filling variant) starts with a transparent volume and fills it. It differs from [Generalized Voxel Coloring](5526-generalized-voxel-coloring.md) by relying on an ordinal visibility constraint to simplify visibility checks, allowing for a single-scan reconstruction.
- **When to choose**: When reconstructing textured surfaces from multi-view images where depth information is available or can be estimated.
- **Link**: [Voxel Coloring](4418-voxel-coloring.md)

### Reference Reconstruction
#### definition
Reconstructs the original voxel grid from exported component bricks, handling overlaps by taking the maximum value [S1, unofficial].

#### step
Load metadata, initialize result and coverage arrays of shape dimensions_xyz[::-1] (ZYX) with uint8 zeros [S1, unofficial]. For each component record, read the .raw file into block, then upsample block along each axis by repeating values spacing times (from comp['spacing_xyz'][::-1]) [S1, unofficial]. Crop block to extent_xyz[::-1], map to result at origin_xyz[::-1] using `np.maximum(result[target], block, out=result[target])`, and update `coverage[target] += block > 0` [S1, unofficial].

#### pitfall
Overlapping bricks are resolved by `np.maximum`, which may cause lossy merging if values differ [S1, unofficial].

## Background
The term "Voxel Coloring" was introduced by Seitz and Dyer in 1997 [S1, unofficial]. The method was developed to address the limitations of earlier volumetric reconstruction techniques that struggled with texture consistency and visibility. It has been widely used in computer vision for 3D modeling from image sequences. The "Reference Reconstruction" variant appears in modern volumetric data pipelines for handling tiled or chunked volume data, as seen in tools like VolumeSegmenter [S1, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
