# Preprocessing Pipeline

Also called: VolCon and Compress Pipeline, Volume Preprocessing Pipeline, buildVolumeImage, crop-downsample-convert

## Overview
The Preprocessing Pipeline prepares raw volumetric data for efficient rendering by applying label banding, cropping to the body bounding box, downsampling, adjusting spacing, and optionally converting to 16-bit float before wrapping in VTK `imageData` [S2, unofficial]. It differs from the [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by focusing on global volume normalization and compression rather than spatial partitioning into disjoint regions, and from the [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by not explicitly grouping bricks via intensity bands for WebGL2 optimization [S1]. It serves as a foundational step for multi-resolution rendering systems, often feeding into octree-based compression stages [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw volume data (e.g., DICOM slices, raw binary files), optional label maps, and configuration parameters (`PERF` object).
*   **Output:** A normalized, cropped, and potentially downsampled volume ready for rendering, typically represented as a VTK `imageData` structure or octree bricks [S1][S2, unofficial].

**Data Structures:**
*   **PERF Configuration:**
    *   `crop` (boolean): Enables body bounding box cropping.
    *   `down` (integer): Downsampling factor applied to dimensions and spacing.
    *   `tex` (string): Texture format flag; `'f16'` triggers 16-bit float conversion.
*   **VTK imageData:** Contains dimensions, spacing, origin, and scalar data (voxel values) [S2, unofficial].
*   **Octree Bricks:** If using the VolCon/Compress variant, data is stored as octree brick files [S1].

**Steps:**
1.  **Label Banding (Optional):** If label maps exist, apply `bandLabeledHuValues` to map raw values to labeled Hounsfield Units; otherwise, use raw values [S2, unofficial].
2.  **Cropping (Conditional):** If `PERF.crop` is true:
    *   Compute the body bounding box using an air threshold (`AIR_HU = -500`) [S2, unofficial].
    *   Crop the volume to this bounding box.
    *   Shift the origin to preserve world coordinates [S2, unofficial].
3.  **Downsampling:** Downsample the volume by `PERF.down` factor using `strideDownsample` [S2, unofficial].
4.  **Spacing Adjustment:** Multiply the voxel spacing by `PERF.down` to maintain physical dimensions [S2, unofficial].
5.  **Format Conversion (Conditional):** If `PERF.tex === 'f16'`, convert voxel values to 16-bit float [S2, unofficial].
6.  **Data Structure Construction:** Build a VTK `imageData` object with the new dimensions, adjusted spacing, shifted origin, and processed scalar data [S2, unofficial].
7.  **Mapper Configuration:** If using 16-bit float (`f16`), set the mapper's `preferSizeOverAccuracy` flag to prioritize rendering speed over precision [S2, unofficial].
8.  **Binding:** Bind the processed data to the rendering mapper [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(raw_volume, labels, perf_config):
    """
    Preprocesses a volume for rendering based on configuration.
    
    Args:
        raw_volume: 3D array of voxel values.
        labels: Optional 3D array of label indices.
        perf_config: Dictionary with keys 'crop', 'down', 'tex'.
        
    Returns:
        vtk_image_data: Processed volume data structure.
    """
    # Step 1: Label Banding
    if labels is not None:
        # Apply label banding to map raw values to labeled HU values
        volume = band_labeled_hu_values(raw_volume, labels)
    else:
        volume = raw_volume.copy()

    # Step 2: Cropping
    origin = [0, 0, 0]
    spacing = [1.0, 1.0, 1.0] # Default spacing, assumed to be adjusted later or provided
    
    if perf_config.get('crop', False):
        # Compute body bounding box using air threshold
        air_threshold = -500
        bbox = compute_body_bounding_box(volume, air_threshold)
        
        # Crop volume to bounding box
        volume = crop_to_bbox(volume, bbox)
        
        # Shift origin to preserve world coordinates
        # Note: Exact origin shift logic depends on bbox definition relative to original origin
        origin = adjust_origin_for_crop(bbox, original_origin)

    # Step 3: Downsampling
    down_factor = perf_config.get('down', 1)
    if down_factor > 1:
        volume = stride_downsample(volume, down_factor)
        spacing = [s * down_factor for s in spacing]

    # Step 4: Format Conversion
    tex_format = perf_config.get('tex', 'f32')
    if tex_format == 'f16':
        volume = convert_to_float16(volume)
        prefer_size_over_accuracy = True
    else:
        prefer_size_over_accuracy = False

    # Step 5: Build VTK imageData
    dimensions = volume.shape
    vtk_data = build_vtk_image_data(
        dimensions=dimensions,
        spacing=spacing,
        origin=origin,
        scalars=volume
    )

    # Step 6: Configure Mapper
    if prefer_size_over_accuracy:
        vtk_data.mapper.set_prefer_size_over_accuracy(True)

    # Step 7: Bind Data
    vtk_data.mapper.set_input_data(vtk_data)

    return vtk_data
```

**Parameters:**
*   `AIR_HU`: -500. Threshold value for air outside the body, used to compute the body bounding box [S2, unofficial].
*   `PERF.crop`: Boolean flag enabling body bounding box cropping [S2, unofficial].
*   `PERF.down`: Integer downsampling factor applied to dimensions and spacing [S2, unofficial].
*   `PERF.tex`: String flag; when `'f16'`, triggers 16-bit float conversion and sets mapper size preference [S2, unofficial].
*   Octree depth and maximum level of wavelet compression: User-defined parameters for the VolCon/Compress variant [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Reinterpreting a `Uint8Array` buffer as `Float32Array` is safe in JavaScript engines due to native little-endian architecture, provided metadata validates `byte_order === 'little'`. Using `DataView` for byte-by-byte reading is significantly slower (~15s+ for a 145MB volume) and can cause stale canvas frames during dataset switches [S2, unofficial].
*   **Degenerate Input:** If `PERF.down` is too large, significant detail may be lost. If `PERF.crop` is true but no body is detected (e.g., empty volume), the bounding box computation may fail or return an empty volume [G].

## Subtopics

### [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md)
The VolumeSegmenter Pipeline is an end-to-end method for compressing large RAW volumetric datasets into compact, spatially mapped bricks organized into selectable logical groups, enabling efficient rendering in WebGL2. It reduces stored and GPU-resident voxel data significantly, though it may increase rendering time due to sparse lookup and ray-marching overhead compared to monolithic source rendering. It differs from the [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by explicitly grouping bricks via intensity bands and adjacency rather than purely spatial partitioning, and builds on the general [Preprocessing Pipeline](6733-preprocessing-pipeline.md) framework.

### [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md)
The Volume Partitioning Pipeline is a data preprocessing method that splits a 3D volume into disjoint foreground regions, optionally partitions them spatially or by intensity, crops them to bounding boxes, downsamples, and exports them as raw files with metadata. It solves the problem of managing large 3D datasets by reducing memory footprint and enabling efficient out-of-core or chunked processing. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by focusing on intensity-based foreground extraction and spatial partitioning rather than semantic segmentation, and from [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md) by using a generic intensity-threshold approach rather than seismic-specific physics.

### [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md)
The 3D Seismic Data Compression Pipeline is a lossy compression method for 3D seismic data that processes volumes in 3D blocks (cubes), applying normalization, closed-loop differential pulse-code modulation (DPCM) with quantization, and Deflate compression to reduce file size while preserving trace integrity. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) and [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by operating on fixed 3D spatial-temporal blocks rather than semantic segments or arbitrary partitions, and from the legacy [**.sdc v2 Trace Compression Pipeline**](#-sdc-v2-trace-compression-pipeline) by using 3D predictive coding instead of 1D trace-level encoding.

### Slice Preprocessing
Slice Preprocessing is a preprocessing phase used for creating cropped slices to be distributed across the network for rendering. Each raw data slice is read once. Non-empty regions are segmented using a mask volume, cropped and written to disc memory. Out-of-core region growing is used if no mask has been provided. Concurrently, data extent information is consolidated into a series of slab-projection slices [S4, unofficial].

### Wild Scene Preprocessing & Test Set Selection
Wild Scene Preprocessing & Test Set Selection is a preprocessing pipeline that filters out low-quality images and images dominated by transient objects to ensure robust training and evaluation on unconstrained real-world datasets [S5, unofficial].
**Steps:**
1.  Filter training images: Remove images with NIMA score < 3 or transient object occupancy > 80% (computed via DeepLab v3) [S5, unofficial].
2.  Select test images: Choose images with NIMA score in the 70th percentile and transient occluder occupancy in the bottom 25%, then manually verify to minimize transient occluders [S5, unofficial].
**Parameters:**
*   NIMA threshold: < 3 for removal [S5, unofficial].
*   Transient occupancy threshold: > 80% for removal, bottom 25% percentile for test selection [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** The preprocessing time depends on the volume size, downsampling factor, and cropping complexity. For the VolCon/Compress variant, deeper octrees increase preprocessing time but improve clipping accuracy [S1].
*   **Memory Complexity:** Downsampling and cropping reduce the memory footprint. The VolCon/Compress variant stores data as octree bricks, which can be more compact than monolithic volumes but may require more file accesses for accurate subvolume approximation [S1].
*   **Measured Performance:** Preprocessing time for various datasets with an octree depth of two and two levels of wavelet compression was measured on a Core2Duo laptop with 2GB RAM [S1]. Specific times are not given in the sources.
*   **Strengths:** Reduces data size, improves rendering performance by focusing on relevant regions (cropping), and enables efficient multi-resolution rendering (octree/wavelet) [S1][S2, unofficial].
*   **Limitations:** May increase rendering time due to sparse lookup and ray-marching overhead if the volume is highly sparse (e.g., after aggressive cropping or segmentation) [S1]. The VolCon/Compress variant may require more file accesses for accurate subvolume approximation [S1].
*   **When to Use:** When dealing with large volumetric datasets that need to be rendered efficiently, especially in web-based or distributed environments [S1][S4, unofficial].
*   **When Not to Use:** When real-time, high-precision rendering of the entire volume is required without compression artifacts or latency from sparse lookups [S1].

## Background
The Preprocessing Pipeline originates from the need to handle large-scale volumetric data for distributed rendering systems. The VolCon and Compress Pipeline was introduced in the context of gigascale multiresolution volume rendering on distributed display clusters [S1]. The crop-downsample-convert approach is used in reinforcement learning-based volume rendering transfer function studies [S2, unofficial]. General volume preprocessing techniques, including filtering, interpolation, and segmentation, are discussed in the context of acceleration techniques for volume rendering [S3, unofficial]. Slice preprocessing is part of massive data management for distributed volume visualization [S4, unofficial]. Wild scene preprocessing is used in unconstrained pose-prior-free neural radiance fields research [S5, unofficial].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
[S2] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
[S3] 14 Acceleration Techniques for Volume Rendering Mark W. Jones 14 Acceleration Techniques for Volume Rendering Mark W. Jones, https://docslib.org/doc/3014834/14-acceleration-techniques-for-volume-rendering-mark-w-jones, fetched 2026-10-28 (unofficial)
[S4] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S5] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
