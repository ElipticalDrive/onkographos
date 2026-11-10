# Preprocessing Pipeline

Also called: VolCon and Compress Pipeline, Volume Preprocessing Pipeline

## Overview
The Preprocessing Pipeline is a multi-stage method for transforming raw volumetric slice data into a compressed, multi-resolution representation suitable for efficient rendering on distributed or resource-constrained systems [S1]. It solves the problem of high memory bandwidth and storage requirements for large-scale volumes by applying spatial subdivision (octrees) and frequency-domain compression (wavelets) [S1]. It differs from general [Compression](4-compression.md) by explicitly structuring the workflow into distinct stages of spatial indexing followed by spectral encoding, and from [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by focusing on hierarchical brick-based storage rather than simple geometric cropping [S1][S2, unofficial].

## How it works
The pipeline processes input slice files to produce a set of compressed octree brick files. The process is divided into two primary stages, often preceded by data-specific cleanup steps.

**Inputs:**
*   Raw volumetric data, typically stored as a sequence of 2D slice files (e.g., DICOM, TIFF) [S1][S2, unofficial].
*   User-defined parameters: Octree depth ($D$) and maximum wavelet compression level ($L$) [S1].

**Outputs:**
*   A set of files containing octree bricks with embedded wavelet coefficients [S1].

**Data Structures:**
*   **Octree Brick:** A node in the octree hierarchy. Each brick stores a subset of the volume. At leaf nodes, data is stored as wavelet coefficients. The structure allows for spatial locality and multi-resolution access [S1].
*   **Wavelet Coefficients:** Transformed values representing the volume data in the frequency domain, enabling lossy compression by discarding high-frequency details [S1].

**Steps:**

1.  **Pre-filtering and Interpolation (Optional/Conditional):**
    *   If the input data contains noise, apply spatial filters such as median, modal, or k-closest averaging filters to remove artifacts [S2, unofficial].
    *   If the resolution differs significantly between axes (e.g., Z-axis resolution is much lower than XY), interpolate new slices to create a regular 3D volume with equal dimensions in each axis [S2, unofficial].

2.  **Stage 1: Octree Construction:**
    *   Read the input slice files.
    *   Compute the octree structure using a tool (e.g., VolCon) [S1].
    *   The algorithm recursively subdivides the volume space. The depth of the octree is controlled by the parameter $D$ [S1].
    *   Store the resulting spatial hierarchy as "octree brick files" [S1].
    *   *Note:* Deeper octrees ($D$ increases) generate smaller brick files, improving the accuracy of subvolume approximation for specific tiles, but increase the number of file accesses required during rendering [S1].

3.  **Stage 2: Wavelet Compression:**
    *   Input the octree bricks from Stage 1.
    *   Calculate the wavelet representation for each brick using a tool (e.g., Compress) [S1].
    *   Apply wavelet transform to the data within each brick.
    *   Compress the coefficients up to the maximum level $L$ defined by the user [S1].
    *   Store the final compressed bricks [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_pipeline(input_slices, octree_depth, max_wavelet_level):
    """
    Executes the two-stage preprocessing pipeline.
    
    Args:
        input_slices: List of raw 2D slice files.
        octree_depth: Integer, maximum depth of the octree.
        max_wavelet_level: Integer, maximum level of wavelet compression.
        
    Returns:
        List of compressed octree brick files.
    """
    # Step 1: Optional Pre-processing
    # Check if interpolation is needed based on axis resolution
    if resolution_z < resolution_xy:
        volume_3d = interpolate_slices(input_slices, target_resolution=resolution_xy)
    else:
        volume_3d = stack_slices(input_slices)
        
    # Optional filtering
    volume_3d = apply_median_filter(volume_3d)
    
    # Step 2: Stage 1 - Octree Construction
    # Compute octree structure from the 3D volume
    octree_bricks = compute_octree(volume_3d, max_depth=octree_depth)
    
    # Step 3: Stage 2 - Wavelet Compression
    compressed_bricks = []
    for brick in octree_bricks:
        # Apply wavelet transform to the brick data
        wavelet_coeffs = wavelet_transform(brick.data)
        
        # Compress coefficients up to the specified level
        # This typically involves thresholding or quantizing high-frequency coefficients
        compressed_data = compress_wavelet(wavelet_coeffs, max_level=max_wavelet_level)
        
        # Create the final brick object
        final_brick = Brick(
            spatial_bounds=brick.spatial_bounds,
            data=compressed_data,
            depth=brick.depth
        )
        compressed_bricks.append(final_brick)
        
    return compressed_bricks
```

**Parameters:**
*   `octree_depth`: Controls the granularity of the spatial subdivision. Higher values lead to smaller bricks but more overhead [S1].
*   `max_wavelet_level`: Controls the extent of frequency-domain compression. Higher values allow for greater compression ratios but may introduce visible artifacts [S1].

**Edge Cases and Pitfalls:**
*   **Resolution Mismatch:** If Z-resolution is significantly lower than XY-resolution without interpolation, the resulting volume will be anisotropic, potentially causing artifacts in ray-marching [S2, unofficial].
*   **File Access Overhead:** Increasing octree depth reduces brick size but increases the number of file I/O operations during rendering. This trade-off must be balanced based on storage speed [S1].
*   **Noise:** Raw medical data often contains noise. Failure to filter may result in the octree or wavelet transform capturing noise as significant detail, reducing compression efficiency [S2, unofficial].

## Subtopics

### [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md)
The VolumeSegmenter Pipeline is an end-to-end method for compressing large RAW volumetric datasets into compact, spatially mapped bricks organized into selectable logical groups, enabling efficient rendering in WebGL2. It reduces stored and GPU-resident voxel data significantly, though it may increase rendering time due to sparse lookup and ray-marching overhead compared to monolithic source rendering. It differs from the [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by explicitly grouping bricks via intensity bands and adjacency rather than purely spatial partitioning, and builds on the general [Preprocessing Pipeline](6733-preprocessing-pipeline.md) framework.

### [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md)
The Volume Partitioning Pipeline splits a 3D volume into foreground regions, optionally partitions them spatially or by intensity, crops them to bounding boxes, downsamples, and exports them as raw files with metadata. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by focusing on geometric and intensity-based partitioning of foreground data for export rather than semantic segmentation, and from [Preprocessing Pipeline](6733-preprocessing-pipeline.md) by specifically handling the decomposition and packaging of volume components.

### Slice Preprocessing
A preprocessing phase used for creating cropped slices to be distributed across the network for rendering. Each raw data slice is read once. Non-empty regions are segmented using a mask volume, cropped and written to disc memory. Out-of-core region growing is used if no mask has been provided. Concurrently, data extent information is consolidated into a series of slab-projection slices [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The preprocessing time depends on the volume size, octree depth, and wavelet levels. The sources provide measured times for specific hardware (Core2Duo laptop, 2GB RAM) but do not give a general formula [S1].
*   **Memory Complexity:** The octree structure requires memory proportional to the number of nodes, which is influenced by the octree depth [S1].
*   **Performance:** Deeper octrees improve the accuracy of subvolume approximation for rendering tiles but increase file access overhead [S1].
*   **Strengths:** Enables multi-resolution rendering and efficient storage of large datasets [S1].
*   **Limitations:** The two-stage process adds complexity. The trade-off between brick size and file access overhead must be carefully managed [S1].

## Background
The Preprocessing Pipeline was developed for giga-scale multiresolution volume rendering on distributed display clusters [S1]. It was introduced to handle the challenges of rendering large volumetric datasets by combining spatial subdivision (octrees) with frequency-domain compression (wavelets) [S1]. The pipeline was implemented using tools named VolCon and Compress [S1].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
[S2] 14 Acceleration Techniques for Volume Rendering Mark W. Jones 14 Acceleration Techniques for Volume Rendering Mark W. Jones, https://docslib.org/doc/3014834/14-acceleration-techniques-for-volume-rendering-mark-w-jones, fetched 2026-10-28 (unofficial)
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
