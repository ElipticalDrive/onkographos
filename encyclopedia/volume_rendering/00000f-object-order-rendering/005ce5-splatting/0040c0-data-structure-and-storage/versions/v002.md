# Data Structure and Storage


## Overview
This family defines the memory layouts, loading interfaces, and indexing strategies required to store volumetric primitives (splats) for efficient GPU rasterization [S1, unofficial]. It solves the problem of transferring large-scale point-cloud or Gaussian data from CPU memory to GPU textures while maintaining fast access for rendering and picking [S1, unofficial]. It differs from sibling articles by focusing on the storage and loading mechanics rather than the rendering pipeline, anti-aliasing, or sorting algorithms [S1, unofficial].

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [G]. All methods in this family store primitive data in GPU-accessible buffers or textures, load them asynchronously to prevent main-thread blocking, and maintain separate indexing structures to decouple source data from display parameters [S1, unofficial]. Rendering transformations and compositing are handled by the parent [Splatting](23-splatting.md).

## Subtopics

### [Seed Generation for Initial Splats](4634-seed-generation-for-initial-splats.md)
Generates initial splat primitives from volumetric data using gradient-based and uniform grid sampling to balance high-frequency detail capture with smooth region coverage. Chosen when an optimal, sparse set of splat positions is required without excessive redundancy.

### [Splat Texture Layout & Mapping](7089-splat-texture-layout-mapping.md)
Defines the mechanism for storing and accessing splat primitive data within GPU-accessible 3D textures, enabling efficient hardware-accelerated rasterization. Sets itself apart by organizing per-primitive data into structured texture records that leverage GPU hardware filtering and caching.

### [Splat Data Structure (ISplatSource)](10289-splat-data-structure-isplatsource.md)
An abstract interface designed for the asynchronous loading of 3D Gaussian Splat data, returning a structured `LoadResult` upon completion. Distinguished by its non-blocking callback mechanism and specific handling of spherical harmonic coefficient packing for GPU consumption.

### [World-aligned lattice octree bake](11637-world-aligned-lattice-octree-bake.md)
Constructs a hierarchical level-of-detail (HLOD) octree for Gaussian splats aligned to world axes, grouping splats into leaves with bounded counts and ordering them by importance. Differs from standard splatting by pre-computing a spatial hierarchy and bounding volumes to prevent false resident culls and optimize memory access.

### Indexed Splats

### definition
- [S1, unofficial] An abstract class extending `Splats` that provides fixed packed storage with a visible-index contract for rendering and CPU reads, separating source opacity from display fade.

### layout
- [S1, unofficial] `IndexedSplatsOptions`: capacity (number), layerSize (number), numSh (number), blockBits (number), packedOpacityGroups (boolean, optional).
- [S1, unofficial] `IndexedSplats` fields: layerSize (readonly), numSh (readonly 0-4), opacities (SplatOpacityTable), sourceIndices (Uint32Array), visibleIndices (Uint32Array), sourceArrays, sourceCenters, sourceTextures, indexTexture, blockBits (readonly), disposed, uploadQueue (PendingUploads), spatialChunks, spatialBoundsBytes.

### comparison
- [S1, unofficial] `IndexedSplats` differs from `Splats` by maintaining separate `sourceIndices` and `visibleIndices` arrays, allowing opacity fading for display without altering source data used in picking.

## Background
The data structures and loading interfaces described here originate from the `Gaussian-Splat-Lite` repository, an unofficial implementation of 3D Gaussian Splatting optimized for web-based rendering environments [S1, unofficial]. The architecture was developed to address performance bottlenecks in loading and storing large Gaussian splat clouds in JavaScript, leveraging Web Workers for asynchronous I/O and WebGL textures for GPU storage [S1, unofficial].

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
