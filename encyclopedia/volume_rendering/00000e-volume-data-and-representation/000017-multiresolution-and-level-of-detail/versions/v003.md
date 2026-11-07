# Multiresolution and level of detail


## Overview
Multiresolution and level of detail (LOD) techniques provide a hierarchical framework for interpreting volumetric data, allowing the system to use the most economical resolution for each feature based on spatial hierarchy and available memory bandwidth [S1, unofficial][S2, unofficial]. This family solves the problem of rendering large-scale volumetric datasets interactively by replacing high-resolution data with lower-resolution approximations for distant or less important regions, thereby reducing GPU memory usage and rendering latency [S2, unofficial]. It is distinguished from sibling methods like [Compression](4-compression.md) (which focuses on storage efficiency) and [Dense grids](3-dense-grids-and-memory-layout.md) (which focus on memory layout) by explicitly organizing data into multiple resolution levels that can be selected or blended during rendering [S2, unofficial].

## Shared principles
The core mechanism of this family involves generating a hierarchy of data representations at different spatial resolutions [S2, unofficial]. Common mathematical and structural operations include:

1.  **Scaling Factors:** Defining an integer or float scale factor $s$ such that the new resolution $N_{new}$ relates to the previous resolution $N_{prev}$ by $N_{new} = s \times N_{prev}$ or $N_{new} / N_{prev} = s$ [S1, unofficial][S5, unofficial].
2.  **Hierarchical Structures:** Using spatial structures (like Octrees or Bricks) or frequency-domain transforms (like Wavelets) to represent data at multiple scales [S2, unofficial].
3.  **Interpolation and Filtering:** Using interpolation or filtering kernels to estimate values at new grid points or to generate coarser levels from finer ones [S1, S2, unofficial].
4.  **Selection and Culling:** Dynamically choosing which resolution level to render based on spatial hierarchy (distant or less important regions) and hardware constraints [S1, S2, unofficial].

## Subtopics

### [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md)
Reduces voxel grid resolution by a specified integer factor using max pooling. It is distinguished by its simplicity and deterministic nature, relying on block-wise maximum operations rather than learned parameters or complex geometric decompositions. Choose this when a fast, lossy (for MIP) downsampling of regular grids is required.

### [Mesh Simplification](1032-mesh-simplification.md)
Reduces triangle count of surface meshes via iterative edge collapses, typically using Quadric Error Metrics (QEM). It differs from siblings by operating on explicit polygonal meshes rather than voxel grids, serving as a post-processing step for surfaces derived from volumetric data. Choose this when rendering extracted isosurfaces rather than the volume itself.

### [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md)
Encodes volumetric data into a multiresolution wavelet coefficient hierarchy, allowing for on-demand or block-wise reconstruction during rendering. It solves high memory bandwidth requirements by exploiting spatial coherence and sparsity in the frequency domain. Choose this for efficient lossy/lossless compression and adaptive resolution selection based on error metrics.

### [Multi-Tetra Framework](1183-multi-tetra-framework.md)
Approximates regular volume data using tetrahedra of varying detail levels, generated through recursive subdivision and stored in a binary tree. It solves the crack problem between LODs by ensuring subdivision uses existing vertex indices without interpolation. Choose this for smooth, crack-free LOD transitions in regular grids.

### [Morphological Pyramid](1641-morphological-pyramid.md)
Uses morphological erosion/dilation combined with dyadic downsampling to create a hierarchy of approximations and detail signals. It exploits the commutativity of MIP with morphological dilation to allow coarse approximations during interaction. Choose this for interactive MIP rendering with progressive refinement.

### [Multi-Resolution Volume Visualization (Split-and-Collapse)](1692-multi-resolution-volume-visualization-split-and-collapse.md)
Dynamically adjusts resolution of volume areas based on memory bandwidth and user interaction using a greedy split/collapse algorithm. Unlike static LOD, it continuously updates the representation during interaction. Choose this for interactive exploration where hardware constraints vary.

### [Level-of-Detail Volume Rendering via 3D Textures](1706-level-of-detail-volume-rendering-via-3d-textures.md)
Provides adaptive LOD selection for multi-resolution rendering using 3D texture mapping, guaranteeing consistent interpolation and fixing boundary artifacts via polygon clipping. Choose this for GPU-accelerated rendering of large datasets with smooth LOD transitions.

### [Streaming MIP Implementation (Basic)](1803-streaming-mip-implementation-basic.md)
Approximates MIP by projecting pre-sorted detail coefficients from a morphological pyramid directly to the frame buffer. It streams coefficients by L1-error contribution, allowing progressive refinement. Choose this for real-time MIP with limited memory bandwidth.

### [Multiresolution Decomposed Volume Rendering](2277-multiresolution-decomposed-volume-rendering.md)
A general framework for rendering decomposed volume data (e.g., Wavelet/Fourier coefficients) directly without full reconstruction. It supports adaptive rendering and error bounds checking. Choose this for progressive rendering and error-bounded visualization of transformed data.

### [Downscaled Grid Map Generation](2338-downscaled-grid-map-generation.md)
Creates a low-resolution coordinate grid and an index map to map high-resolution coordinates to low-resolution locations. It differs from [Mipmap Generation](2602-mipmap-generation.md) by focusing on coordinate indexing rather than value aggregation. Choose this for differentiable rendering pipelines requiring coordinate transformations.

### [Level of Detail for Volumetric Terrain](2459-level-of-detail-for-volumetric-terrain.md)
A fast, GPU-friendly LOD algorithm for real-time rendering of volumetric terrain using arbitrary density functions. It generates geometry on the fly, avoiding preprocessing costs. Choose this for procedural or large-scale terrain visualization.

### [Mipmap Generation](2602-mipmap-generation.md)
Creates a hierarchy of downscaled volume data to enable multi-resolution rendering and faster traversal. It explicitly organizes data into a structured mip-chain accessible via texture sampling. Choose this as a standard preprocessing step for texture-based volume rendering.

### [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-raycasting-with-bricking.md)
Combines spatial decomposition (bricking) with intensity-based transfer function mapping (window/level) to accelerate visualization of large medical datasets. It differs from [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) by using spatial tiling rather than frequency-domain compression. Choose this for cache-friendly raycasting with dynamic transfer functions.

### [Active Cut Management](3380-active-cut-management.md)
Maintains a set of nodes whose cells cover the model space exactly once, balancing visual quality and frame time. Choose this for managing active regions in adaptive rendering.

### [LOD Selection and Culling](5338-lod-selection-and-culling.md)
Algorithms for selecting the optimal LOD and culling low-detail regions to save bandwidth. Choose this for optimizing data management, streaming, and hysteresis when switching resolutions.

### [Upsampling and Super-Resolution](5339-upsampling-and-super-resolution.md)
Addresses increasing spatial resolution to recover fine details or correct discrete positioning errors. Distinguished from siblings by its goal of generating higher-fidelity representations from lower-resolution inputs. Choose this for detail recovery or performance optimization via upscaling.

### [Wavelet-based Multiresolution](6266-wavelet-based-multiresolution.md)
Uses wavelet transforms to decompose volume data into frequency subbands for multiresolution representation. Choose this for frequency-domain compression and error-bounded traversal.

### [Adaptive Resolution Data Management](8131-adaptive-resolution-data-management.md)
Manages data at varying resolutions to optimize memory usage and rendering performance. Choose this for dynamic resource allocation in large-scale visualization.

### [Adaptive-FAM](8156-adaptive-fam.md)
A functional approximation multi-resolution representation that uses adaptive encoding to dynamically compress micro-blocks based on their content, designed for out-of-core interactive visualization of large-scale volumetric datasets. Choose this for memory-constrained environments requiring content-aware compression.

#### Patch Merging Layer
Downsamples spatial resolution by half while doubling channels by concatenating four overlapping spatial patches [S3, unofficial].
- **Step:** Verify input dimensions are even (`assert H % 2 == 0 and W % 2 == 0`). Reshape to spatial: `x = x.reshape(B, H, W, C)`. Extract four overlapping patches: `x0 = x[:, 0::2, 0::2, :]`, `x1 = x[:, 1::2, 0::2, :]`, `x2 = x[:, 0::2, 1::2, :]`, `x3 = x[:, 1::2, 1::2, :]`. Concatenate along channel dimension: `x = concatenate_along_last_dim([x0, x1, x2, x3])`. Reshape to sequence: `x = x.reshape(B, -1, 4 * C)`. Apply normalization: `x = self.norm(x)`. Apply linear reduction to halve channels: `x = self.reduction(x)` [S3, unofficial].
- **Parameter:** `input_resolution` (tuple of H, W), `dim` (input channels), `norm_layer` (default `nn.LayerNorm`) [S3, unofficial].
- **Layout:** Input tensor `x`: shape `(B, H*W, C)`. Output tensor `x`: shape `(B, H/2*W/2, 2*dim)` [S3, unofficial].
- **Code:**
  ```python
  def patch_merging(x, H, W, C):
      assert H % 2 == 0 and W % 2 == 0
      x_spatial = x.reshape(B, H, W, C)
      x0 = x_spatial[:, 0::2, 0::2, :]
      x1 = x_spatial[:, 1::2, 0::2, :]
      x2 = x_spatial[:, 0::2, 1::2, :]
      x3 = x_spatial[:, 1::2, 1::2, :]
      # Concatenate along channel dimension
      x_concat = concatenate_along_last_dim([x0, x1, x2, x3])
      # Reshape to sequence
      new_H, new_W = H // 2, W // 2
      x_seq = x_concat.reshape(B, new_H * new_W, 4 * C)
      return x_seq
  ```
  [S3, unofficial]

#### iNGP Hierarchy Configuration
Uses 10 grid scales spaced by powers of 2 from 16 to 8192, with 4 channels per level. Grid sizes $n_l > 128$ are parameterized using a hash of size $128^3$. This configuration works slightly better and faster than iNGP's similar approach (2 channels, spacing by $\sqrt{2}$) [S4].

#### World Dimension Constants and Hierarchy
Defines constants for a specific implementation: `BRICK_DIM`: 4, `BRICK_VOXELS`: 64. `WORLD_BRICKS_X`: 128, `WORLD_BRICKS_Y`: 64, `WORLD_BRICKS_Z`: 128 (Total: 1,048,576 bricks). `WORLD_VOXELS_X`: 512, `WORLD_VOXELS_Y`: 256, `WORLD_VOXELS_Z`: 512. `WORLD_TILES_X`: 32, `WORLD_TILES_Y`: 16, `WORLD_TILES_Z`: 32 (Total: 16,384 tiles). `WORLD_CHUNKS_X`: 8, `WORLD_CHUNKS_Y`: 4, `WORLD_CHUNKS_Z`: 8 (Total: 256 chunks). `WORLD_L4_X`: 2, `WORLD_L4_Y`: 1, `WORLD_L4_Z`: 2 (Total: 4 L4 nodes). `PROBE_SPACING`: 8 voxels. `PROBE_DIM_X`: 64, `PROBE_DIM_Y`: 32, `PROBE_DIM_Z`: 64 (Total: 131,072 probes). `STORAGE_CHUNK_BRICKS`: 8. `STORAGE_CHUNK_VOXELS`: 32. `WORLD_STORE_CX`: 16, `WORLD_STORE_CY`: 8, `WORLD_STORE_CZ`: 16 (Total: 2,048 storage chunks) [S5, unofficial].

#### Parallel Multiresolution
A parallel multiresolution technique proposed by Gao et al. to accelerate large dataset visualization [S6].

## Background
Multiresolution representations in image processing and computer vision provide a simple hierarchical framework for interpreting image information [S1, unofficial]. At different resolutions, details generally characterize different physical structures of the scene, allowing the most economical resolution to be used for each feature [S1, unofficial]. Large volumetric data rendering algorithms rely on hierarchical data structures to improve processing, with Octrees and Wavelet transformations being the most popular methods to transform original data into multi-resolution representations [S2, unofficial]. These methods allow low-resolution data to replace original data during rendering to reduce GPU memory usage, with minimal impact on visual quality for distant regions [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S3] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S4] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S5] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S6] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S7] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
[S8] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
