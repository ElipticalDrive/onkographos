# Screen-space voxel techniques

Also called: Screen-space ambient occlusion and halos

## Overview
Screen-space voxel techniques are a family of real-time methods that compute ambient occlusion and shape-enhancing halos by analyzing depth discontinuities and neighbor relationships in screen space [S1, unofficial]. They set themselves apart from traditional volume rendering by operating on opaque structures within the volume (internal halos) rather than just background edges, and from standard G-buffer AO by using voxel-based occupancy or summed area tables to avoid haloing artifacts and allow interactive parameter tuning [S1, unofficial].

## Shared principles
All techniques in this family operate on screen-space geometry data (typically a depth map or G-buffer) to approximate local illumination effects without full volumetric ray marching. They rely on the analysis of depth variations to identify geometric features such as crevices, edges, and internal structures. The core mechanism involves sampling the neighborhood of a pixel to determine occlusion factors or halo intensities, which are then blended with the base shading (e.g., Phong) to enhance shape perception. Key shared concepts include the use of a vicinity window (defined by $size_x, size_y$) to gather neighbor information, the calculation of average depth or occupancy within that window, and the application of a scaling factor to modulate the intensity of the effect.

## Subtopics

### [Screen-space Halos (Internal)](14271-screen-space-halos-internal.md)
Screen-space Halos (Internal) renders halos over non-background pixels by analyzing depth discontinuities and using average depth to modulate halo intensity. It sets itself apart from basic halo generation by operating on opaque structures within the volume rather than just the background, and from ambient occlusion methods by providing shape cues through color blending rather than just darkening. It builds on [Local illumination, shadows and ambient occlusion](34-local-illumination-shadows-and-ambient-occlusion.md).

### [Voxel-Space Ambient Occlusion](21187-voxel-space-ambient-occlusion.md)
Voxel-Space Ambient Occlusion (VSAO) is a multiresolution, screen-space technique that computes ambient occlusion using a coarse voxel representation of scene geometry. It improves upon standard G-buffer based techniques by avoiding artifacts such as haloing and is scalable to the level-of-detail desired by the user. Unlike screen-space techniques that rely on depth comparisons or sphere approximations, VSAO assumes voxels are small objects and computes occlusion using an area rendering equation formulation.

### Screen-space Ambient Occlusion and Halos (Interactive Control)
#### definition
A real-time screen-space method where ambient occlusion and halos are computed from scratch each frame, allowing interactive parameter changes without performance cost [S1, unofficial].

#### step
Compute average depths using Summed Area Tables (SAT) to allow changing halo size ($size_x, size_y$) with no extra cost (same number of texture accesses) [S1, unofficial].

#### parameter
- $size_x$ and $size_y$: Resolution of the vicinity region used to compute the occlusion factor, adjustable to fine-tune results [S1, unofficial].
- $Color_{vic}$: The emphasizing color for depth variations, adjustable if depth variations need to be accentuated differently [S1, unofficial].
- $factor_{vic}$: A factor controlling the perceptibility of shading modification in ambient occlusion; higher values make the modification more perceptible [S1, unofficial].

#### pitfall
Fixed functions for ambient occlusion may excessively darken certain regions while barely modifying others if depth complexity varies across the volume [S1, unofficial].

## Background
The techniques originate from research into depth-enhanced volume visualization, specifically aiming to improve the perception of internal structures and shape cues in real-time rendering. The foundational work describes a screen-space method for computing ambient occlusion and halos using depth maps and summed area tables, published in a dissertation in 2026 [S1, unofficial]. The Voxel-Space Ambient Occlusion variant references earlier work by Eisemann and Decorat for screen-space voxelization [S1, unofficial].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
