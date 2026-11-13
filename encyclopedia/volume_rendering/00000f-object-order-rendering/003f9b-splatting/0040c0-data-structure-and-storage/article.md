# Data Structure and Storage

Also called: Splat Texture Layout & Mapping, ISplatSource, Indexed Splats

## Overview
This family defines the memory layouts, loading interfaces, and indexing strategies required to store volumetric primitives (splats) for efficient GPU rasterization. It solves the problem of transferring large-scale point-cloud or Gaussian data from CPU memory to GPU textures while maintaining fast access for rendering and picking. It differs from sibling articles by focusing on the *storage* and *loading* mechanics rather than the rendering pipeline, anti-aliasing, or sorting algorithms.

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [G].

1.  **Primitive Representation:** The volume is decomposed into primitives [G]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6, unofficial][S10, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6, unofficial][S10, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane [G]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6, unofficial][S10, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
4.  **Compositing:** Contributions are accumulated to form the final image [G]. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7, unofficial]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].

## Subtopics

### [Splat Texture Layout & Mapping](7089-splat-texture-layout-mapping.md)
Maps per-primitive splat data into two 3D textures backed by `Uint32Array` buffers for efficient GPU sampling. Each splat corresponds to two 16-byte texture records within this layout. It is chosen when high-throughput texture sampling is required for large splat counts, requiring careful management of capacity mismatches and 3D coordinate paging. See [Splat Texture Layout & Mapping](7089-splat-texture-layout-mapping.md).

### [Splat Data Structure (ISplatSource)](10289-splat-data-structure-isplatsource.md)
An abstract interface designed to load splat cloud data asynchronously, decoupling file I/O from the rendering thread to prevent UI freezes. It differs from synchronous loading methods by returning a `LoadResult` object only upon completion, allowing the caller to manage memory and error states explicitly. It serves as the primary entry point for populating the GPU-ready splat buffers described in [Splat Texture Layout & Mapping](7089-splat-texture-layout-mapping.md). See [Splat Data Structure (ISplatSource)](10289-splat-data-structure-isplatsource.md).

### Indexed Splats
An abstract class extending `Splats` that provides fixed packed storage with a visible-index contract for rendering and CPU reads, separating source opacity from display fade [S1, unofficial]. Its layout is defined by `IndexedSplatsOptions` (capacity, layerSize, numSh, blockBits, packedOpacityGroups) and fields including `opacities`, `sourceIndices`, `visibleIndices`, `sourceArrays`, `sourceCenters`, `sourceTextures`, `indexTexture`, `blockBits`, `disposed`, `uploadQueue`, `spatialChunks`, and `spatialBoundsBytes` [S1, unofficial]. It differs from standard `Splats` by maintaining separate `sourceIndices` and `visibleIndices` arrays, allowing opacity fading for display without altering source data used in picking [S1, unofficial].

## Background
The specific data structures and loading interfaces described here originate from the `Gaussian-Splat-Lite` project, an unofficial implementation of 3D Gaussian Splatting optimized for web-based rendering [S1, unofficial]. The architecture was developed to address the performance bottlenecks of loading and storing large Gaussian splat clouds in JavaScript environments, leveraging Web Workers for asynchronous I/O and WebGL textures for GPU storage.

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
