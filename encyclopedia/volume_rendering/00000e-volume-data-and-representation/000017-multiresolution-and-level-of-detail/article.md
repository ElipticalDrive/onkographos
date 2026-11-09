# Multiresolution and level of detail

Also called: Multiresolution representations, hierarchical volume rendering, LOD for volumes

## Overview
Multiresolution and level of detail (LOD) techniques represent volumetric data at multiple spatial resolutions to optimize rendering performance and memory usage [S2, unofficial]. By allowing low-resolution data to replace high-resolution data for distant or less important regions, these methods reduce the amount of data residing in GPU memory without severely affecting rendering quality [S2, unofficial]. Unlike sibling methods that focus on static storage or compression, this family dynamically selects resolution levels based on view-dependent criteria such as distance, screen footprint, or error metrics [S1, S2, unofficial].

## Shared principles
All methods in this family share the core mechanism of generating a hierarchy of data representations at different spatial resolutions [S2, unofficial]. This hierarchy is typically constructed using one of two approaches:
1.  **Spatial Hierarchies:** Structures like Octrees or Bricks that partition the volume domain into sub-regions of varying sizes [S2, unofficial].
2.  **Frequency-Domain Transforms:** Methods like Wavelets that decompose data into coefficients representing different spatial frequencies [S2, unofficial].

Common mathematical and structural operations include:
*   **Scaling Factors:** Defining a scale factor $s$ such that the new resolution $N_{new}$ relates to the previous resolution $N_{prev}$ by $N_{new} = s \times N_{prev}$ or $N_{new} / N_{prev} = s$ [S1, S5, unofficial].
*   **Interpolation and Filtering:** Using interpolation kernels (e.g., trilinear) or filtering to estimate values at new grid points or to generate coarser levels from finer ones [S1, S2, unofficial].
*   **Selection and Culling:** Dynamically choosing which resolution level to render based on spatial hierarchy (distant or less important regions) and hardware constraints [S1, S2, unofficial].

## Subtopics

### [Mesh Simplification](1032-mesh-simplification.md)
Reduces triangle count of surface meshes via edge collapses. Distinct from frequency-domain methods like Wavelets or tetrahedral frameworks by operating on triangular topology.

### [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md)
Encodes data into frequency-domain coefficients (DWT) for on-demand reconstruction. Preserves spatial locality in the frequency domain, unlike simple downsampling.

### [Multi-Tetra Framework](1183-multi-tetra-framework.md)
Approximates regular volume data using tetrahedra of varying LODs in a binary tree. Solves crack problems by using existing vertex indices without interpolation.

### [Multi-Resolution Volume Visualization (Split-and-Collapse)](1692-multi-resolution-volume-visualization-split-and-collapse.md)
Interactive method using a greedy algorithm to split/collapse bricks based on error metrics and memory constraints. Dynamically updates representation frame-to-frame.

### [Multiresolution Decomposed Volume Rendering](2277-multiresolution-decomposed-volume-rendering.md)
General framework for rendering decomposed data (Fourier, Wavelet, DCT) directly without full reconstruction. Supports progressive refinement and error bounds.

### [iNGP Hierarchy Configuration](2404-ingp-hierarchy-configuration.md)
Configures hash grids for neural radiance fields. Uses 10 levels, 4 channels, and powers-of-2 scale spacing to balance memory and speed.

### [Level of Detail for Volumetric Terrain](2459-level-of-detail-for-volumetric-terrain.md)
GPU-friendly LOD for real-time volumetric terrain. Generates geometry on the fly using hexahedral blocks ($16^3$) and marching cubes, avoiding preprocessing costs.

### [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-raycasting-with-bricking.md)
Combines spatial bricking with window/level intensity mapping. Uses CPU-side brick traversal and explicit window/level mapping rather than hardware mipmaps.

### [Active Cut Management](3380-active-cut-management.md)
View-dependent LOD using an "active cut" of octree nodes. Maintains exactly one coverage of model space, merging/splitting nodes based on screen-space error metrics.

### [LOD Selection and Culling](5338-lod-selection-and-culling.md)
Family of techniques optimizing performance by selecting resolution for visible subregions and culling invisible ones. Operates on volumetric structures (bricks, octrees) rather than meshes.

### [Upsampling and Super-Resolution](5339-upsampling-and-super-resolution.md)
Increases spatial resolution via interpolation or learning to mitigate aliasing. Focuses on upscaling rather than downsampling or simplification.

### [Level-of-Detail Selection](6114-level-of-detail-selection.md)
Uses image-based quality metrics to guide LOD selection, ensuring higher detail only where necessary. Distinguishes itself from purely geometric culling strategies.

### [Wavelet-based Multiresolution](6266-wavelet-based-multiresolution.md)
Decomposes data into spatial-frequency components using wavelets. Preserves high-frequency details at coarse resolutions, unlike spatial averaging.

### [Adaptive Resolution Data Management](8131-adaptive-resolution-data-management.md)
Organizes data into multi-resolution structures based on LOD rather than strict spatial location. Groups blocks by resolution level to reduce storage, introducing interpolation challenges.

### [Adaptive-FAM](8156-adaptive-fam.md)
Out-of-core pipeline using functional approximation with Normalized Control Points (NCP). Minimizes model size while maintaining accuracy, optimized for I/O caching.

### [Pyramid and Downsampling Structures](9527-pyramid-and-downsampling-structures.md)
Generates hierarchy of decreasing resolutions. Operates on spatial grid structures, preserving voxel-based nature, unlike frequency-domain methods.

### [Multiresolution Volume Sampling: Intrablock](11149-multiresolution-volume-sampling-intrablock.md)
Handles sampling within single multiresolution blocks, managing boundaries and resolution differences.

### World Dimension Constants and Hierarchy
*   **Definition:** Specific parameterization for a hierarchical volume representation system [S3, unofficial].
*   **Parameters:**
    *   `BRICK_DIM`: 4. `BRICK_VOXELS`: 64 ($4^3$) [S3, unofficial].
    *   `WORLD_BRICKS_X`: 128, `WORLD_BRICKS_Y`: 64, `WORLD_BRICKS_Z`: 128. Total: 1,048,576 bricks [S3, unofficial].
    *   `WORLD_VOXELS_X`: 512, `WORLD_VOXELS_Y`: 256, `WORLD_VOXELS_Z`: 512 [S3, unofficial].
    *   `WORLD_TILES_X`: 32, `WORLD_TILES_Y`: 16, `WORLD_TILES_Z`: 32. Total: 16,384 tiles [S3, unofficial].
    *   `WORLD_CHUNKS_X`: 8, `WORLD_CHUNKS_Y`: 4, `WORLD_CHUNKS_Z`: 8. Total: 256 chunks [S3, unofficial].
    *   `WORLD_L4_X`: 2, `WORLD_L4_Y`: 1, `WORLD_L4_Z`: 2. Total: 4 L4 nodes. Covers $256^3$ voxel region per bit [S3, unofficial].
    *   `PROBE_SPACING`: 8 voxels. `PROBE_DIM_X`: 64, `PROBE_DIM_Y`: 32, `PROBE_DIM_Z`: 64. Total: 131,072 probes [S3, unofficial].
    *   `STORAGE_CHUNK_BRICKS`: 8. `STORAGE_CHUNK_VOXELS`: 32. `WORLD_STORE_CX`: 16, `WORLD_STORE_CY`: 8, `WORLD_STORE_CZ`: 16. Total: 2,048 storage chunks [S3, unofficial].

### Parallel Multiresolution
*   **Definition:** A parallel multiresolution technique proposed by Gao et al. to accelerate large dataset visualization [S4].
*   **Step:** Proposes a parallel multiresolution technique to accelerate large dataset visualization [S4].

### Importance-based MIP Compositing
*   **Definition:** A compositing model that selects the current valid level of sparseness based on the associated volume object's importance [S5, unofficial].
*   **Step:** Determine the valid level of sparseness for the volume object [S5, unofficial].

### Dynamic Isosurface and Level of Detail (DYNAISOLOD)
*   **Step:** Represent different LODs as differently sized hexahedral "virtual voxels" [S6, unofficial]. Join adjacent voxel cells in the volumetric data grid to create larger virtual voxels [S6, unofficial]. Store the resolution value as a third dimension of the kd-tree [S6, unofficial]. Create the LODTree file (3D kd-tree with associated data) in parallel using MPI [S6, unofficial]. At visualization time, execute search, geometry-creation, and rendering algorithms [S6, unofficial].
*   **Layout:** LODTree file: 3D kd-tree with associated data, where the third dimension encodes resolution/LOD level [S6, unofficial].
*   **Comparison:** Operates in voxel space rather than geometry space, avoiding the assumption of stable underlying surface geometry and eliminating the need for recalculation when thresholds change [S6, unofficial].
*   **Usage:** Core component of Sandia's EIGENNR project; applied to structured rectilinear grid data from the CTH simulation code [S6, unofficial].

## Background
Multiresolution representations provide a hierarchical framework for interpreting image information, where details at different resolutions characterize different physical structures [S1, unofficial]. This approach allows the most economical resolution to be used for each feature, making hierarchical structures more explicit [S1, unofficial]. In volume rendering, hierarchical data structures (spatial or frequency domain) are used to represent data at different resolutions, allowing low-resolution data to replace high-resolution data for distant or less important regions to reduce GPU memory usage [S2, unofficial]. Spatial hierarchies (like Octrees) explore spatial structure, while frequency domain analysis (like Wavelets) transforms data into coefficients for multi-resolution representation [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S3] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S4] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S5] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S6] Dynamic isosurface extraction and level-of-detail in voxel space, https://digital.library.unt.edu/ark:/67531/metadc707496/?q=marching cubes, fetched 2026-10-28 (unofficial)
