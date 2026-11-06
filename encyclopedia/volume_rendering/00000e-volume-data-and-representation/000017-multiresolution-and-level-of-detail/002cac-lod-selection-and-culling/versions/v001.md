# LOD Selection and Culling

Also called: Level-of-Detail Selection, Hierarchical Culling, Streaming LOD

## Overview
LOD Selection and Culling is a family of techniques that optimize volume rendering performance by dynamically selecting the appropriate data resolution based on viewer distance, projected footprint, or importance, while discarding invisible regions via spatial culling [S1]. It distinguishes itself from sibling methods by focusing on the *decision logic* and *data management* (streaming, caching, hysteresis) required to switch between resolutions, rather than the mathematical generation of the multiresolution hierarchy itself (covered in [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md)).

## Shared principles
The core mechanism of this family involves generating a hierarchy of data representations at different spatial resolutions. Common mathematical and structural operations include:

1.  **Scaling Factors:** Defining an integer or float scale factor $s$ such that the new resolution $N_{new}$ relates to the previous resolution $N_{prev}$ by $N_{new} = s \times N_{prev}$ or $N_{new} / N_{prev} = s$ [S1][S5].
2.  **Hierarchical Structures:** Using spatial structures (like Octrees or Bricks) or frequency-domain transforms (like Wavelets) to represent data at multiple scales [S2, unofficial].
3.  **Interpolation and Filtering:** Using bilinear, bicubic, or quadratic spline kernels to estimate values at new grid points or to generate coarser levels from finer ones [S1, S3, unofficial].
4.  **Selection and Culling:** Dynamically choosing which resolution level to render based on error metrics, viewer distance, or hardware constraints [S1, S2, unofficial].

## Subtopics

### [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md)
This method accelerates volume rendering by combining a spatial hierarchy for visibility culling with a multi-resolution data structure for approximation. It solves the performance bottleneck of rendering large volumetric datasets by discarding invisible subregions and rendering visible ones at reduced resolution. It is distinguished from sibling methods by its explicit use of hardware-assisted cell projection for exploration-level rendering and its reliance on bounding volume tests against the view frustum to manage memory transfer and generation. See [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md).

### [Streaming and Level-of-Detail (LOD) selection](3087-streaming-and-level-of-detail-lod-selection.md)
Streaming and Level-of-Detail (LOD) selection is a method for visualizing massive volumetric scenes by adaptively loading data based on viewpoint position, bounding memory usage to visible data and its required "useful" resolution rather than the total scene size. It sets itself apart from static LOD approaches by incrementally updating the data structure to maintain navigation fluidity, avoiding the reload of all data upon viewpoint changes. Unlike [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) which may rely on pre-computed hierarchical structures, this method uses additive ladders to paint a coarse scene from the first chunk and sharpen it as more data arrives, combined with substitutive levels and spatial tiles to reduce the cost of distant or off-screen objects. See [Streaming and Level-of-Detail (LOD) selection](3087-streaming-and-level-of-detail-lod-selection.md).

### [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md)
A method to determine the appropriate level-of-detail for each brick based on the distance of the brick's center to a user-defined focus point. See [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md).

### [Multi-resolution LOD Volume Rendering with Temporal Coherence](5717-multi-resolution-lod-volume-rendering-with-temporal-coherenc.md)
A volume rendering technique that assigns different resolution levels to subvolumes based on their importance (projection area, gaze distance) to optimize performance while maintaining visual quality, using temporal coherence to prevent flickering during view changes. See [Multi-resolution LOD Volume Rendering with Temporal Coherence](5717-multi-resolution-lod-volume-rendering-with-temporal-coherenc.md).

### [Face Subsampling](6025-face-subsampling.md)
See [Face Subsampling](6025-face-subsampling.md).

### [Flat blocking multi-resolution volume rendering](6428-flat-blocking-multi-resolution-volume-rendering.md)
A strategy to load large volume data by dividing the volume into many sub-blocks with equal size but with different resolution or LOD (level of detail) according to the sub-block's importance. See [Flat blocking multi-resolution volume rendering](6428-flat-blocking-multi-resolution-volume-rendering.md).

### Level-of-Detail Selection
#### definition
- An algorithm using an image-based quality metric to select levels of detail for rendering large volumetric data and improving image quality [S1].

## Background
The concept of Level-of-Detail (LOD) in volume rendering emerged alongside the need to render large volumetric datasets interactively. Early work focused on generating multiresolution hierarchies. A significant development in LOD *selection* was made by Wang et al., who developed a level-of-detail selection algorithm using an image-based quality metric to improve image quality in GPU-based raycasting pipelines for large volumetric data [S1]. This approach shifted focus from static resolution assignment to dynamic, quality-driven selection. Other related techniques include the generation of high-quality light field representations for novice audiences [S1] and Fourier Domain Volume Rendering which shifts computation to the spatial frequency domain [S1].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
