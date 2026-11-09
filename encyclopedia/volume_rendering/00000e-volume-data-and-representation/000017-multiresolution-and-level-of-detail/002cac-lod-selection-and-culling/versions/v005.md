# LOD Selection and Culling

Also called: Level of Detail Selection and Culling

## Overview
LOD Selection and Culling is the family of methods that dynamically chooses the appropriate spatial resolution for rendering volumetric data based on viewer position, screen footprint, or data importance, while discarding invisible regions to optimize performance [S1][S2, unofficial]. It differs from sibling methods like [Mesh Simplification](1032-mesh-simplification.md) by operating on volumetric primitives (voxels, bricks, splats) rather than polygonal meshes, and from [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) by focusing on the selection logic and culling criteria rather than the specific multiresolution decomposition transform itself.

## Shared principles
The core mechanism of this family involves generating a hierarchy of data representations at different spatial resolutions [S2, unofficial]. Common mathematical and structural operations include:

1.  **Scaling Factors:** Defining an integer or float scale factor $s$ such that the new resolution $N_{new}$ relates to the previous resolution $N_{prev}$ by $N_{new} = s \times N_{prev}$ or $N_{new} / N_{prev} = s$ [S1][S5, unofficial].
2.  **Hierarchical Structures:** Using spatial structures (like Octrees or Bricks) or frequency-domain transforms (like Wavelets) to represent data at multiple scales [S2, unofficial].
3.  **Interpolation and Filtering:** Using interpolation or filtering kernels to estimate values at new grid points or to generate coarser levels from finer ones [S1][S2, unofficial].
4.  **Selection and Culling:** Dynamically choosing which resolution level to render based on spatial hierarchy (distant or less important regions) and hardware constraints [S1][S2, unofficial].

## Subtopics

### [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md)
This method accelerates volume rendering by combining spatial culling with multi-resolution data approximation. It solves the performance bottleneck of rendering large volumetric datasets by discarding invisible subregions and rendering visible regions at resolutions proportional to their screen importance. It differs from sibling methods like [Streaming and Level-of-Detail (LOD) selection](3087-streaming-and-level-of-detail-lod-selection.md) by explicitly coupling frustum culling with hardware-assisted cell projection for exploration, and from [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md) by using a global projected size threshold rather than a specific focus point for LOD selection.

### [Streaming and Level-of-Detail (LOD) selection](3087-streaming-and-level-of-detail-lod-selection.md)
Streaming and Level-of-Detail (LOD) selection is a method for visualizing massive scenes that exceed rendering or memory budgets by dynamically selecting data resolution based on viewpoint position and screen footprint. It solves the problem of navigating arbitrarily large datasets without stalling the frame rate or exceeding memory limits by combining additive ladders (progressive detail) with substitutive levels (coarse approximations) and spatial culling. It sets itself apart from its siblings by explicitly managing the temporal aspect of data arrival and resolution selection through a "level-of-detail grammar" that prioritizes visual coverage preservation during streaming, whereas methods like [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) focus on static hierarchy traversal, and [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md) relies on fixed spatial bricks rather than splat-based footprints.

### [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md)
Focus Point Based Brick LOD Selection is a method to determine the appropriate level-of-detail (LOD) for each brick in a multiscale volume representation based on the distance of the brick's center to a user-defined focus point. It differs from error-driven siblings like [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) because it relies exclusively on geometric proximity to the focus point, ignoring approximation error. This allows for arbitrary texture resolutions per brick, unlike methods with strict texture-to-geometry correspondence.

### [Choosing Alternative Resolutions](5314-choosing-alternative-resolutions.md)
This method selects a cache-resident resolution level for a volume rendering brick when the desired resolution is not available in memory. It leverages a bitmap of partially mapped resolutions stored in each residency-octree node to efficiently identify candidate levels without random searching. The approach prioritizes lower resolutions before attempting higher ones, iterating through candidates until a mapped page is found or all options are exhausted.

### [Finding Higher Resolution Levels](5315-finding-higher-resolution-levels.md)
This method determines the next higher resolution level in a residency octree traversal when the current node is only partially mapped at the current resolution. It solves the problem of efficiently skipping resolution levels that are fully unmapped or already processed, ensuring that the renderer accesses the finest available data that still contains partial occupancy information. It is distinct from sibling methods like [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md) which generate data, or [LOD Selection and Culling](5338-lod-selection-and-culling.md) which make final rendering decisions; this method is a specific bitwise lookup operation used during hierarchical traversal to find the next candidate resolution index.

### [Multi-resolution LOD Volume Rendering with Temporal Coherence](5717-multi-resolution-lod-volume-rendering-with-temporal-coherenc.md)
Multi-resolution LOD Volume Rendering with Temporal Coherence is a technique that assigns different resolution levels to subvolumes based on their visual importance (such as projection area, gaze distance, or viewer proximity) to optimize rendering performance while maintaining visual quality. It distinguishes itself from standard LOD methods by explicitly addressing the visual instability caused by frequent resolution changes during view updates, using a temporal coherence mechanism to prevent flickering. Unlike [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) which may rely on strict hierarchical traversal, this method allows arbitrary texture resolutions for subvolumes provided level transitions are constrained.

### [Face Subsampling](6025-face-subsampling.md)
Face Subsampling is a Level-of-Detail (LOD) technique for tetrahedral volume rendering that dynamically selects a subset of mesh faces to render, balancing visual fidelity against frame rate constraints. It solves the problem of interactive rendering on large tetrahedral meshes by reducing the number of rasterized primitives without pre-aggregating data into voxels or bricks. It differs from sibling methods like [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) by operating directly on the unstructured mesh topology rather than a hierarchical grid, and from [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md) by being primarily face-centric rather than brick-centric.

### [Level-of-Detail Selection](6114-level-of-detail-selection.md)
Level-of-Detail (LOD) selection is a method for rendering large volumetric datasets by dynamically choosing the appropriate resolution level for different regions of the scene to balance image quality and performance. It solves the problem of rendering constraints on large data by using an image-based quality metric to guide the selection of detail levels, thereby improving perceived image quality without exceeding hardware budgets. Unlike sibling methods that may rely on spatial culling or fixed hierarchical traversal, this approach explicitly optimizes for visual fidelity relative to the selected primitive count.

### [Flat blocking multi-resolution volume rendering](6428-flat-blocking-multi-resolution-volume-rendering.md)
Flat blocking multi-resolution volume rendering is a strategy for loading and rendering large volume datasets by dividing the volume into sub-blocks of equal physical size but assigning each block a different resolution (Level of Detail, LOD) based on its internal data variation. It solves the problem of memory bandwidth and rendering time for large volumes by avoiding uniform high-resolution storage. Unlike hierarchical structures that change spatial extent, this method keeps block dimensions constant while varying the voxel count within them. It is distinguished from its siblings by its explicit use of a "flat" blocking structure where LOD is determined per block via a variation heuristic, often implemented using min/max octrees.

### [Level of Detail (LoD) for Large-Scale Scenes](7325-level-of-detail-lod-for-large-scale-scenes.md)
Level of Detail (LoD) for Large-Scale Scenes is a technique that regulates the amount of detail needed to represent large-scale scenes by precomputing several simplified versions of the dataset. It solves the problem of rendering massive volumes efficiently by selecting the most appropriate resolution level at runtime based on criteria such as distance to the viewer. Unlike continuous LOD methods, this approach uses discrete, static representations, which are easier to precompute and impose very little overhead at runtime, making it highly suitable for graphics hardware that favors display lists or vertex arrays.

### [GPU-Based Dynamic LOD Calculation](10845-gpu-based-dynamic-lod-calculation.md)
This method calculates Level-of-Detail (LOD) levels for visible tiles entirely on the GPU using compute shaders, replacing traditional CPU-side computation based on viewpoint distance and screen-space error. It distinguishes itself by integrating terrain ruggedness into the LOD decision logic and performing the calculation in parallel for thousands of tiles, thereby reducing CPU overhead.

### LoD Selection via Projected Width
#### step
- [S1] Load multiple computed LoD volumes simultaneously (one per LoD level).
- [S1] Choose which volume to lookup for data at each voxel based on the required LoD level.
- [S1] Determine the required LoD level using a metric based on projected width on the camera, following usual PathTracer MipMap selection.

### Pyramid Level Model Selection
#### definition
- [S2, unofficial] Determines the level-of-detail (LOD) hierarchy model for streaming volume bricks based on dataset metadata and user policy.
#### step
- [S2, unofficial] If policy is 'native', return metadata.pyramid directly.
- [S2, unofficial] If policy is 'legacy' or unset, return undefined to trigger fallback to a uniform 2:1 virtual pyramid.
- [S2, unofficial] Fallback occurs when native levels do not fit the native model (e.g., a coarse level is a few voxels short of half its parent).
#### parameter
- [S2, unofficial] pyramid: 'native' for per-axis factors from metadata, 'legacy' for uniform 2:1 virtual pyramid. Unset enables automatic fallback.
#### pitfall
- [S2, unofficial] Native model fails if a coarse level is slightly smaller than half its parent's dimensions; triggers legacy fallback unless explicitly disabled.

## Background
The concept of Level of Detail (LOD) in volume rendering originated from the need to handle large datasets interactively. Early work focused on hierarchical structures like octrees and brick-based representations to enable out-of-core rendering. The specific techniques described in this family, such as focus-point-based selection and temporal coherence, have been developed to address artifacts like "popping" and visual instability during camera movement. Modern implementations increasingly leverage GPU compute shaders for dynamic LOD calculation and streaming architectures for massive datasets [S1][S2, unofficial].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
[S2] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
