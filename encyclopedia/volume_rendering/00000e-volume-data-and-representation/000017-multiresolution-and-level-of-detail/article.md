# Multiresolution and level of detail

Also called: Multiresolution representations, hierarchical volume rendering, LOD for volumes

## Overview
Multiresolution and level of detail (LOD) techniques represent volumetric data at multiple spatial resolutions to optimize rendering performance and memory usage [S2, unofficial]. By allowing low-resolution data to replace high-resolution data for distant or less important regions, these methods reduce the amount of data residing in GPU memory without severely affecting rendering quality [S2, unofficial]. Unlike sibling methods that focus on static storage or compression, this family dynamically selects resolution levels based on view-dependent criteria such as distance, screen footprint, or error metrics [S1, unofficial][S2, unofficial].

## Shared principles
All methods in this family share the core mechanism of generating a hierarchy of data representations at different spatial resolutions [S2, unofficial]. This hierarchy is typically constructed using one of two approaches:
1. **Spatial Hierarchies:** Structures like Octrees or Bricks that partition the volume domain into sub-regions of varying sizes [S2, unofficial].
2. **Frequency-Domain Transforms:** Methods like Wavelets that decompose data into coefficients representing different spatial frequencies [S2, unofficial].

Common mathematical and structural operations include:
* **Scaling Factors:** Defining a scale factor $s$ such that the new resolution $N_{\text{new}}$ relates to the previous resolution $N_{\text{prev}}$ by $N_{\text{new}} = s \times N_{\text{prev}}$ or $N_{\text{new}} / N_{\text{prev}} = s$ [LOD Selection and Culling](5338-lod-selection-and-culling.md).
* **Interpolation and Filtering:** Using interpolation kernels (e.g., trilinear) or filtering to estimate values at new grid points or to generate coarser levels from finer ones [LOD Selection and Culling](5338-lod-selection-and-culling.md).
* **Selection and Culling:** Dynamically choosing which resolution level to render based on spatial hierarchy (distant or less important regions) and hardware constraints [LOD Selection and Culling](5338-lod-selection-and-culling.md).

## Subtopics

### [Mesh Simplification](1032-mesh-simplification.md)
Reduces triangle count of surface meshes via edge collapses. Sets itself apart from frequency-domain methods like Wavelets or tetrahedral frameworks by operating directly on triangular topology. Choose it when interactive rotation speed is prioritized over full-resolution isosurface extraction.

### [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md)
Encodes data into frequency-domain coefficients (DWT) for on-demand reconstruction. Sets itself apart from simple downsampling by preserving spatial locality in the frequency domain. Choose it when memory bandwidth and storage reduction are critical.

### [Multi-Tetra Framework](1183-multi-tetra-framework.md)
Approximates regular volume data using tetrahedra of varying LODs in a binary tree. Sets itself apart from fixed grid resolutions by solving crack problems using existing vertex indices without interpolation. Choose it when smooth resolution changes across areas are needed without adding new sample points.

### [Multi-Resolution Volume Visualization (Split-and-Collapse)](1692-multi-resolution-volume-visualization-split-and-collapse.md)
Interactive method using a greedy algorithm to split/collapse bricks based on error metrics and memory constraints. Sets itself apart from static LOD approaches by dynamically updating the representation frame-to-frame. Choose it for real-time interactive rendering of large datasets within hardware limits.

### [Multiresolution Decomposed Volume Rendering](2277-multiresolution-decomposed-volume-rendering.md)
General framework for rendering decomposed data (Fourier, Wavelet, DCT) directly without full reconstruction. Sets itself apart from Fourier/Wavelet methods that require full reconstruction by allowing direct rendering from coefficients. Choose it when adaptive rendering and efficient coding are required without full data decompression.

### [iNGP Hierarchy Configuration](2404-ingp-hierarchy-configuration.md)
Configures hash grids for neural radiance fields. Sets itself apart from the original iNGP by using 4 channels per level and powers-of-2 scale spacing. Choose it for optimizing neural rendering convergence and performance.

### [Level of Detail for Volumetric Terrain](2459-level-of-detail-for-volumetric-terrain.md)
GPU-friendly LOD for real-time volumetric terrain. Sets itself apart from bottom-up techniques by generating geometry on the fly, avoiding preprocessing costs. Choose it for vast, complex procedural terrain without heightmaps.

### [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-raycasting-with-bricking.md)
Combines spatial bricking with window/level intensity mapping. Sets itself apart from frequency-domain methods by using a spatial brick layout. Choose it for accelerating large medical datasets on standard PC hardware.

### [Active Cut Management](3380-active-cut-management.md)
View-dependent LOD using an "active cut" of octree nodes. Sets itself apart from static multiresolution methods by maintaining exactly one coverage of model space and adapting resolution in real-time. Choose it when balancing visual fidelity against frame time and memory bandwidth constraints.

### [LOD Selection and Culling](5338-lod-selection-and-culling.md)
Family of techniques optimizing performance by selecting resolution for visible subregions and culling invisible ones. Sets itself apart from mesh-based methods by operating on volumetric structures (bricks, octrees). Choose it for rendering large volumetric datasets that exceed hardware budgets.

### [Upsampling and Super-Resolution](5339-upsampling-and-super-resolution.md)
Increases spatial resolution via interpolation or learning to mitigate aliasing. Sets itself apart from siblings by focusing on *increasing* resolution rather than decreasing it. Choose it to enhance detail or prepare data for downstream tasks.

### [Level-of-Detail Selection](6114-level-of-detail-selection.md)
Uses image-based quality metrics to guide LOD selection, ensuring higher detail only where necessary. Sets itself apart from purely geometric or memory-based culling strategies. Choose it when image-based quality optimization is prioritized.

### [Wavelet-based Multiresolution](6266-wavelet-based-multiresolution.md)
Decomposes data into spatial-frequency components using wavelets. Sets itself apart from spatial hierarchies by operating in the frequency domain, allowing mathematically rigorous error bounds. Choose it for smooth interpolation between resolutions.

### [Adaptive Resolution Data Management](8131-adaptive-resolution-data-management.md)
Organizes data into multi-resolution structures based on LOD rather than strict spatial location. Sets itself apart from methods maintaining strict spatial locality by grouping blocks based on content metrics. Choose it when content metrics (e.g., variance) can drive resolution assignment.

### [Adaptive-FAM](8156-adaptive-fam.md)
Out-of-core pipeline using functional approximation with Normalized Control Points (NCP). Sets itself apart from frequency decomposition methods by using spatial partitioning and adaptive compression. Choose it for interactive out-of-core visualization with reduced caching time.

### [Pyramid and Downsampling Structures](9527-pyramid-and-downsampling-structures.md)
Generates hierarchy of decreasing resolutions. Sets itself apart from frequency-domain methods by operating directly on spatial grid structures. Choose it for optimizing rendering performance and memory usage on spatial grids.

### [Multiresolution Volume Sampling: Intrablock](11149-multiresolution-volume-sampling-intrablock.md)
Handles sampling within single multiresolution blocks, managing boundaries and resolution differences. Sets itself apart from Interblock sampling by restricting lookup to a single spatial region. Choose it when consistent sampling within a defined spatial partition is required without blending neighboring data.

### World Dimension Constants and Hierarchy
* **Definition:** Specific parameterization for a hierarchical volume representation system [S3, unofficial].
* **Parameters:**
  * `BRICK_DIM`: 4. `BRICK_VOXELS`: 64 ($4^3$) [S3, unofficial].
  * `WORLD_BRICKS_X`: 128, `WORLD_BRICKS_Y`: 64, `WORLD_BRICKS_Z`: 128. Total: 1,048,576 bricks [S3, unofficial].
  * `WORLD_VOXELS_X`: 512, `WORLD_VOXELS_Y`: 256, `WORLD_VOXELS_Z`: 512 [S3, unofficial].
  * `WORLD_TILES_X`: 32, `WORLD_TILES_Y`: 16, `WORLD_TILES_Z`: 32. Total: 16,384 tiles [S3, unofficial].
  * `WORLD_CHUNKS_X`: 8, `WORLD_CHUNKS_Y`: 4, `WORLD_CHUNKS_Z`: 8. Total: 256 chunks [S3, unofficial].
  * `WORLD_L4_X`: 2, `WORLD_L4_Y`: 1, `WORLD_L4_Z`: 2. Total: 4 L4 nodes. Covers $256^3$ voxel region per bit [S3, unofficial].
  * `PROBE_SPACING`: 8 voxels. `PROBE_DIM_X`: 64, `PROBE_DIM_Y`: 32, `PROBE_DIM_Z`: 64. Total: 131,072 probes [S3, unofficial].
  * `STORAGE_CHUNK_BRICKS`: 8. `STORAGE_CHUNK_VOXELS`: 32. `WORLD_STORE_CX`: 16, `WORLD_STORE_CY`: 8, `WORLD_STORE_CZ`: 16. Total: 2,048 storage chunks [S3, unofficial].

### Parallel Multiresolution
* **Definition:** A parallel multiresolution technique proposed by Gao et al. to accelerate large dataset visualization [S4].
* **Step:** Proposes a parallel multiresolution technique to accelerate large dataset visualization [S4].

### Importance-based MIP Compositing
* **Definition:** A compositing model that selects the current valid level of sparseness based on the associated volume object's importance [S5, unofficial].
* **Step:** Determine the valid level of sparseness for the volume object [S5, unofficial].

### Dynamic Isosurface and Level of Detail (DYNAISOLOD)
* **Step:** Represent different LODs as differently sized hexahedral "virtual voxels" [S7, unofficial]. Join adjacent voxel cells in the volumetric data grid to create larger virtual voxels [S7, unofficial]. Store the resolution value as a third dimension of the kd-tree [S7, unofficial]. Create the LODTree file (3D kd-tree with associated data) in parallel using MPI [S7, unofficial]. At visualization time, execute search, geometry-creation, and rendering algorithms [S7, unofficial].
* **Layout:** LODTree file: 3D kd-tree with associated data, where the third dimension encodes resolution/LOD level [S7, unofficial].
* **Comparison:** Operates in voxel space rather than geometry space, avoiding the assumption of stable underlying surface geometry and eliminating the need for recalculation when thresholds change [S7, unofficial].
* **Usage:** Core component of Sandia's EIGENNR project; applied to structured rectilinear grid data from the CTH simulation code [S7, unofficial].

## Background
Multiresolution representations provide a hierarchical framework for interpreting image information, where details at different resolutions characterize different physical structures [S1, unofficial]. This approach allows the most economical resolution to be used for each feature, making hierarchical structures more explicit [S1, unofficial]. In volume rendering, hierarchical data structures (spatial or frequency domain) are used to represent data at different resolutions, allowing low-resolution data to replace high-resolution data for distant or less important regions to reduce GPU memory usage [S2, unofficial]. Spatial hierarchies (like Octrees) explore spatial structure, while frequency domain analysis (like Wavelets) transforms data into coefficients for multi-resolution representation [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S3] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S4] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S5] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S6] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S7] Dynamic isosurface extraction and level-of-detail in voxel space, https://digital.library.unt.edu/ark:/67531/metadc707496/?q=marching cubes, fetched 2026-10-28 (unofficial)
