# Anisotropic Voxel Representation and Filtering

Also called: Anisotropic Pre-Integration

## Overview
Anisotropic Voxel Representation and Filtering addresses the "two red-green wall problem" in voxel cone tracing, where isotropic averaging of opaque voxels with different colors or opacities in upper octree levels causes light leaking by mixing materials as if they were semi-transparent [S1, S4]. It stores six directional copies of color and alpha (along ±X, ±Y, ±Z axes) instead of a single isotropic average, allowing the cone tracer to detect directional occlusion (e.g., thin walls) and preventing light leaks through thin geometry [S3, unofficial][S4]. Unlike its siblings, it explicitly preserves directional visibility information during mipmap generation to mitigate the artifacts inherent in standard isotropic pre-integration [S3, unofficial].

## Shared principles
The core mechanism replaces the single scalar value per voxel in non-leaf octree nodes with an array of six directional values corresponding to the major axes: $+X, -X, +Y, -Y, +Z, -Z$ [S1, unofficial]. This representation prevents color mixing and opacity under-estimation during MIP-mapping by integrating values along specific axes, allowing for accurate interpolation of lighting at render time [S1, unofficial].

**Mipmap Generation (Pre-Integration):**
For each non-leaf voxel, the six directional values are computed by performing a step of volumetric integration in depth for each direction [S1, S4]. This involves compositing the 8 child voxels front-to-back along the specific axis [S3, unofficial]. The results are averaged to produce the final directional value for the parent voxel [S1, S4].

**Runtime Sampling:**
During cone tracing, the voxel value is retrieved by linearly interpolating the values from the three closest directions with respect to the view direction $\vec{v}$ [S4]. The 3 axis-matching directional textures are blended weighted by the squared dot product of the trace direction and the axis direction:
$$ weight = \text{dot}(\text{direction}, \text{axis})^2 $$
[S3, unofficial][S4].

**Memory and Scope:**
Directional values are only computed and used for voxels not located in the last octree level (not at full resolution) [S4]. Leaf nodes store only the standard single value [S1, unofficial]. Storing directional values for all properties increases memory consumption by 1.5x compared to isotropic [S4]. The base level of the mipmap is identical for all directions and can be shared, reducing the cost from 6x to 1.5x [S5, unofficial]. The total storage overhead is cited as 1.75x in some contexts [S1, unofficial].

## Subtopics

### [Voxel Cone Tracing Anisotropic Filtering](2552-voxel-cone-tracing-anisotropic-filtering.md)
[Voxel Cone Tracing Anisotropic Filtering](2552-voxel-cone-tracing-anisotropic-filtering.md) addresses the "two red-green wall problem" in voxel cone tracing, where isotropic averaging of opaque voxels with different colors or opacities in upper octree levels causes light leaking by mixing materials as if they were semi-transparent. It stores six directional copies of color and alpha (along ±X, ±Y, ±Z axes) instead of a single isotropic average, allowing the cone tracer to detect directional occlusion (e.g., thin walls) and preventing light leaks through thin geometry. Unlike its siblings, it explicitly preserves directional visibility information during mipmap generation to mitigate the artifacts inherent in standard isotropic pre-integration.

### [Anisotropic Voxel Representation](4963-anisotropic-voxel-representation.md)
[Anisotropic Voxel Representation](4963-anisotropic-voxel-representation.md) is a data storage format for voxel cone tracing that stores directional pre-integration values to prevent color mixing and opacity under-estimation during MIP-mapping. It solves the "two red-green wall problem" and light leaking artifacts that occur when averaging opaque voxels with different colors or mixed opacity in upper octree levels. Unlike standard isotropic MIP-mapping which averages all neighbors uniformly, this method preserves directional coherence by integrating values along specific axes, allowing for accurate interpolation of lighting at render time.

#### Anisotropic Pre-Integration
- [S1, unofficial] Use the 6 main directions.
- [S1, unofficial] Reconstruct values by linear interpolation of the 3 closest directions.
- [S1, unofficial] Storage overhead multiplier: 1.75x.
- [S1, unofficial] Provides a better approximation of the volume rendering integral compared to isotropic methods, but costs 1.75x the storage requirement and slightly slower sampling.

## Background
The method was proposed to address light leaking artifacts in voxel cone tracing when low resolution mipmap levels are used [S1, unofficial]. It relies on an anisotropic pre-integration of voxel values stored in a direction-dependent way in the structure [S1, unofficial]. The approach builds on the foundational work of Gigavoxels, which introduced ray-guided streaming for efficient and detailed voxel rendering [S1, unofficial].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing: An Insight, https://openalex.org/W4296123939, fetched 2026-10-26 (unofficial)
