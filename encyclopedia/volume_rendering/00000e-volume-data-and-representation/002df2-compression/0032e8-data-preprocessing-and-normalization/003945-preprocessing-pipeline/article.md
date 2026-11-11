# Preprocessing Pipeline

Also called: VolCon and Compress Pipeline, Volume Preprocessing Pipeline, buildVolumeImage, crop-downsample-convert

## Overview
The Preprocessing Pipeline prepares raw volumetric data for efficient rendering by applying label banding, cropping to the body bounding box, downsampling, adjusting spacing, and converting data types before wrapping the result in a standard volume image structure [S2, unofficial]. It differs from general [Volume Normalization](1823-volume-normalization.md) by explicitly handling spatial reduction (cropping/downsampling) and structural organization (octree/wavelet generation) rather than just intensity mapping, and from [Volume Utility Functions](2677-volume-utility-functions.md) by providing a complete end-to-end workflow for data transformation [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw volume data (e.g., DICOM slices, raw binary files), configuration parameters (`PERF` object), and optional label maps.
*   **Output:** A structured volume image object (e.g., `vtkImageData`) containing processed scalar data, adjusted spacing, and origin, ready for rendering or further compression [S2, unofficial].

**Data Structures:**
*   **PERF Configuration:** An object containing:
    *   `crop` (boolean): Enables body bounding box cropping.
    *   `down` (integer): Downsampling factor.
    *   `tex` (string): Texture format flag; `'f16'` triggers 16-bit float conversion.
*   **Volume Metadata:**
    *   `dimensions`: Array of integers $[D_x, D_y, D_z]$.
    *   `spacing`: Array of floats $[S_x, S_y, S_z]$.
    *   `origin`: Array of floats $[O_x, O_y, O_z]$.
    *   `scalars`: Array of numeric values representing voxel intensities.

**Steps:**
1.  **Label Banding:** If label data exists, apply `bandLabeledHuValues` to map raw values to labeled Hounsfield Units; otherwise, use raw values [S2, unofficial].
2.  **Cropping (Optional):** If `PERF.crop` is true:
    *   Compute the body bounding box using an air threshold `AIR_HU = -500` [S2, unofficial].
    *   Crop the volume to this bounding box.
    *   Shift the volume origin to preserve world coordinates relative to the original space [S2, unofficial].
3.  **Downsampling:** Downsample the volume by `PERF.down` factor using a stride-based downsampling method (`strideDownsample`) [S2, unofficial].
4.  **Spacing Adjustment:** Multiply the volume spacing by `PERF.down` to maintain correct physical dimensions [S2, unofficial].
5.  **Data Type Conversion:** If `PERF.tex === 'f16'`, convert the scalar values to 16-bit floating-point format [S2, unofficial].
6.  **Volume Construction:** Build the `vtkImageData` object using the new dimensions, adjusted spacing, adjusted origin, and processed scalars [S2, unofficial].
7.  **Mapper Configuration:** If using 16-bit float (`f16`), set the mapper's `preferSizeOverAccuracy` flag to optimize rendering performance [S2, unofficial].
8.  **Binding:** Bind the processed data to the volume mapper [S2, unofficial].

**Pitfalls:**
*   **Byte Order and Performance:** When reading raw binary data in JavaScript, reinterpreting a `Uint8Array` buffer as a `Float32Array` is safe because JS engines are natively little-endian and metadata validates `byte_order === 'little'`. Using `DataView` for byte-by-byte reading is significantly slower (~15s+ for a 145MB volume) and causes stale canvas frames during dataset switches [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(raw_data, perf_config, label_map=None):
    """
    Preprocesses raw volume data for rendering.
    
    Args:
        raw_data: Array of raw voxel values.
        perf_config: Object with keys 'crop', 'down', 'tex'.
        label_map: Optional array of labels.
        
    Returns:
        volume_image: Structured volume data object.
    """
    # Step 1: Label Banding
    if label_map is not None:
        # Apply banding function to map raw values to labeled HU
        processed_data = band_labeled_hu_values(raw_data, label_map)
    else:
        processed_data = raw_data.copy()

    # Step 2: Cropping
    origin = [0.0, 0.0, 0.0] # Initial origin
    spacing = [1.0, 1.0, 1.0] # Initial spacing (to be updated)
    dimensions = list(processed_data.shape)

    if perf_config.get('crop', False):
        # Compute body bounding box using AIR_HU threshold
        air_threshold = -500
        body_bbox = compute_body_bounding_box(processed_data, air_threshold)
        
        # Crop volume to bounding box
        cropped_data = crop_volume(processed_data, body_bbox)
        processed_data = cropped_data
        
        # Shift origin to preserve world coordinates
        # Note: Exact shift calculation depends on original spacing and bbox coordinates
        origin = adjust_origin_for_crop(body_bbox, original_spacing)

    # Step 3: Downsampling
    down_factor = perf_config.get('down', 1)
    if down_factor > 1:
        processed_data = stride_downsample(processed_data, down_factor)
        dimensions = [d // down_factor for d in dimensions]

    # Step 4: Spacing Adjustment
    # Multiply spacing by down factor to maintain physical size
    # Assuming initial spacing is 1.0 or provided separately
    current_spacing = [s * down_factor for s in spacing]

    # Step 5: Data Type Conversion
    if perf_config.get('tex') == 'f16':
        processed_data = convert_to_float16(processed_data)

    # Step 6: Volume Construction
    volume_image = vtk_image_data(
        dimensions=dimensions,
        spacing=current_spacing,
        origin=origin,
        scalars=processed_data
    )

    # Step 7: Mapper Configuration (if applicable in rendering context)
    if perf_config.get('tex') == 'f16':
        volume_image.mapper.prefer_size_over_accuracy = True

    # Step 8: Binding
    # Bind data to mapper (implementation specific to renderer)
    bind_data_to_mapper(volume_image)

    return volume_image
```

## Subtopics

### [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md)
The VolumeSegmenter Pipeline is an end-to-end method for compressing large RAW volumetric datasets into compact, spatially mapped bricks organized into selectable logical groups, enabling efficient rendering in WebGL2. It reduces stored and GPU-resident voxel data significantly, though it may increase rendering time due to sparse lookup and ray-marching overhead compared to monolithic source rendering. It differs from the [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by explicitly grouping bricks via intensity bands and adjacency rather than purely spatial partitioning, and builds on the general [Preprocessing Pipeline](6733-preprocessing-pipeline.md) framework.

### [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md)
The Volume Partitioning Pipeline splits a 3D volume into foreground regions, optionally partitions them spatially or by intensity, crops them to bounding boxes, downsamples, and exports them as raw files with metadata. It differs from the [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by focusing on explicit component partitioning and compact export rather than semantic segmentation, and from the [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md) by its general-purpose intensity-based foreground detection and flexible partitioning strategies.

### [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md)
The 3D Seismic Data Compression Pipeline is a lossy compression method for 3D seismic data that processes volumes in 3D blocks (cubes), applying normalization, closed-loop differential pulse-code modulation (DPCM) with quantization, and Deflate compression to reduce file size while preserving trace integrity. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) and [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by operating on fixed 3D spatial-temporal blocks rather than semantic segments or arbitrary partitions, and from the legacy [**.sdc v2 Trace Compression Pipeline**](#-sdc-v2-trace-compression-pipeline) by using 3D predictive coding instead of 1D trace-level encoding.

### Slice Preprocessing
Slice Preprocessing is a phase used for creating cropped slices to be distributed across the network for rendering. Each raw data slice is read once. Non-empty regions are segmented using a mask volume, cropped and written to disc memory. Out-of-core region growing is used if no mask has been provided. Concurrently, data extent information is consolidated into a series of slab-projection slices [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The preprocessing time depends on the octree depth and wavelet compression levels. Deeper octrees generate smaller brick files but increase file access overhead [S1].
*   **Memory Complexity:** The method reduces stored and GPU-resident voxel data significantly, particularly when using downsampling and compression [S1].
*   **Performance:** Measured on a Core2Duo laptop with 2GB RAM, preprocessing times vary by dataset size and compression settings [S1].
*   **Strengths:** Efficiently reduces data size for network distribution and rendering; supports multi-resolution structures for adaptive detail [S1][S4, unofficial].
*   **Limitations:** May increase rendering time due to sparse lookup and ray-marching overhead compared to monolithic source rendering [S1].

## Background
The Preprocessing Pipeline originates from research on gigascale multiresolution volume rendering on distributed display clusters [S1]. It was developed to handle large volumetric datasets by breaking them down into manageable octree bricks and applying wavelet compression [S1]. The pipeline has been adapted for various applications, including medical imaging and distributed visualization systems [S2, S3, S4, unofficial].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
[S2] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
[S3] 14 Acceleration Techniques for Volume Rendering Mark W. Jones 14 Acceleration Techniques for Volume Rendering Mark W. Jones, https://docslib.org/doc/3014834/14-acceleration-techniques-for-volume-rendering-mark-w-jones, fetched 2026-10-28 (unofficial)
[S4] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
