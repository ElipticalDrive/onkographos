# Preprocessing Pipeline

Also called: VolCon and Compress Pipeline, Volume Preprocessing Pipeline

## Overview
The Preprocessing Pipeline is a multi-stage method for transforming raw volumetric data (typically slice files) into a compressed, multi-resolution representation suitable for efficient rendering. It solves the problem of high memory bandwidth and storage requirements by applying spatial subdivision and frequency-domain transformation [S1]. It differs from sibling methods like [Sparse structures](5-sparse-structures.md) by explicitly combining octree-based spatial partitioning with wavelet-based spectral compression, and from [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by focusing on algorithmic compression rather than geometric cropping [S1, S1].

## How it works
The pipeline operates in two primary stages, often executed by distinct tools (`VolCon` and `Compress`), preceded by optional data normalization steps [S1][S2, unofficial].

**Inputs and Outputs:**
*   **Input:** Raw volumetric data, typically stored as a set of 2D slice files.
*   **Output:** A set of compressed octree brick files containing wavelet coefficients [S1].

**Data Structures:**
*   **Octree Brick:** A spatial subdivision node. The structure is defined by the octree depth parameter. Deeper octrees result in smaller brick files but require more file accesses during rendering [S1].
*   **Wavelet Coefficients:** The transformed values stored within the bricks. The maximum level of wavelet compression is a user-defined parameter [S1].

**Steps:**

1.  **Pre-filtering and Interpolation (Optional):**
    *   If the input data contains noise, apply spatial filters such as median, modal, or k-closest averaging filters [S2, unofficial].
    *   If the resolution differs significantly between axes (e.g., Z-axis vs. XY-plane), interpolate images to create a 3D volume with equal dimensions in each axis [S2, unofficial].

2.  **Segmentation (Optional):**
    *   Separate objects within the volume using interactive seed points and region growing with thresholds [S2, unofficial].
    *   Adjust segmented regions interactively by eroding incorrectly connected bridges or merging regions that should be connected [S2, unofficial].

3.  **Stage 1: Octree Construction:**
    *   Use the `VolCon` tool to compute the octree structure from the input slice files [S1].
    *   Store the resulting spatial subdivisions as octree brick files [S1].

4.  **Stage 2: Wavelet Compression:**
    *   Use the `Compress` tool to calculate the wavelet representation based on the octree bricks generated in Stage 1 [S1].
    *   Apply wavelet transformation up to the user-defined maximum level [S1].

**Parameters:**
*   **Octree Depth:** User-defined. Controls the granularity of spatial subdivision. Deeper trees yield smaller bricks but increase I/O overhead [S1].
*   **Maximum Wavelet Level:** User-defined. Controls the depth of the frequency domain transformation [S1].

**Edge Cases and Pitfalls:**
*   **I/O Overhead:** Increasing octree depth improves clipping accuracy but increases the number of file accesses required during rendering [S1].
*   **Anisotropy:** Failure to interpolate anisotropic data may result in incorrect volume representation if equal-dimensional axes are required [S2, unofficial].

## Subtopics

### [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md)
An end-to-end method for compressing large RAW volumetric datasets into compact, spatially mapped bricks organized into selectable logical groups, enabling efficient rendering in WebGL2. It reduces stored and GPU-resident voxel data significantly, though it may increase rendering time due to sparse lookup and ray-marching overhead compared to monolithic source rendering. It differs from the [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by explicitly grouping bricks via intensity bands and adjacency rather than purely spatial partitioning, and builds on the general [Preprocessing Pipeline](6733-preprocessing-pipeline.md) framework.

### [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md)
Splits a 3D volume into foreground regions, optionally partitions them spatially or by intensity, crops them to bounding boxes, downsamples, and exports them as raw files with metadata. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by focusing on geometric and intensity-based partitioning of foreground data for export rather than semantic segmentation, and from [Preprocessing Pipeline](6733-preprocessing-pipeline.md) by specifically handling the decomposition and packaging of volume components.

### [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md)
A lossy compression method for 3D seismic data that processes volumes in 3D blocks (cubes), applying normalization, closed-loop differential pulse-code modulation (DPCM) with quantization, and Deflate compression to reduce file size while preserving trace integrity. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) and [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by operating on fixed 3D spatial-temporal blocks rather than semantic segments or arbitrary partitions, and from the legacy **.sdc v2 Trace Compression Pipeline** by using 3D predictive coding instead of 1D trace-level encoding.

### Slice Preprocessing
A preprocessing phase used for creating cropped slices to be distributed across the network for rendering. Each raw data slice is read once. Non-empty regions are segmented using a mask volume, cropped and written to disc memory. Out-of-core region growing is used if no mask has been provided. Concurrently, data extent information is consolidated into a series of slab-projection slices [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time depends on octree depth and wavelet levels. Specific measurements show preprocessing on a Core2Duo laptop with 2GB RAM for datasets with octree depth 2 and two wavelet levels, but exact timings are not provided in the sources [S1].
*   **Memory Complexity:** The method reduces stored voxel data significantly. However, deeper octrees increase the number of file accesses, which can impact performance on systems with slow I/O [S1].
*   **Strengths:** Provides multi-resolution data structures suitable for distributed display clusters and interactive rendering [S1].
*   **Limitations:** Increased clipping accuracy from deeper octrees comes at the cost of more file accesses [S1].

## Background
The method was developed for giga-scale multiresolution volume rendering on distributed display clusters. It was introduced in the context of rendering large anatomical datasets on LCD clusters, where multi-resolution data structures are computed in a two-stage preprocessing step using tools named `VolCon` and `Compress` [S1].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
[S2] 14 Acceleration Techniques for Volume Rendering Mark W. Jones 14 Acceleration Techniques for Volume Rendering Mark W. Jones, https://docslib.org/doc/3014834/14-acceleration-techniques-for-volume-rendering-mark-w-jones, fetched 2026-10-28 (unofficial)
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
