# Preprocessing Pipeline

Also called: VolCon and Compress Pipeline, Volume Preprocessing Pipeline, buildVolumeImage, crop-downsample-convert

## Overview
The Preprocessing Pipeline prepares raw volumetric data for efficient rendering by applying label banding, cropping to the body bounding box, downsampling, adjusting spacing, and optionally converting to 16-bit float before wrapping in VTK `imageData` [S2, unofficial]. It solves the problem of managing large, high-resolution medical datasets by reducing memory footprint and enabling efficient out-of-core or chunked processing [S1]. It differs from the [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by focusing on intensity-based foreground extraction and spatial partitioning rather than semantic segmentation, and from the [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by explicitly grouping bricks via intensity bands and adjacency rather than purely spatial partitioning [S1].

## How it works
The pipeline consists of two distinct phases: a general data normalization/cropping phase (often implemented as `buildVolumeImage`) and a multi-resolution compression phase (VolCon/Compress) [S1][S2, unofficial].

### Inputs and Outputs
- **Input**: Raw volume data (e.g., DICOM slices, NIfTI, or raw binary files), potentially with associated labels.
- **Output**: A VTK `imageData` object containing the processed scalars, or a set of octree brick files with wavelet representations [S1][S2, unofficial].

### Data Structures
1.  **VTK `imageData`**: Contains dimensions, spacing, origin, and scalar data (float32 or float16) [S2, unofficial].
2.  **Octree Brick Files**: Hierarchical data structure storing compressed subvolumes. Defined by octree depth and wavelet compression level [S1].

### Steps

#### Phase 1: Normalization and Cropping (buildVolumeImage)
This phase prepares the raw data for rendering by ensuring consistent resolution and removing background noise [S2, unofficial].

1.  **Label Banding**: If labels exist, apply `bandLabeledHuValues` to raw values; otherwise, use raw values [S2, unofficial].
2.  **Cropping**: If `PERF.crop` is true, compute the body bounding box using an air threshold (`AIR_HU = -500` HU). Crop the volume to this box and shift the origin to preserve world coordinates [S2, unofficial].
3.  **Downsampling**: Downsample the volume by a factor `PERF.down` using `strideDownsample`. This reduces the number of voxels along each axis [S2, unofficial].
4.  **Spacing Adjustment**: Multiply the voxel spacing by `PERF.down` to maintain the correct physical dimensions of the cropped region [S2, unofficial].
5.  **Format Conversion**: If `PERF.tex` is `'f16'`, convert the scalar values to 16-bit float (half-precision) [S2, unofficial].
6.  **VTK Construction**: Build a `vtkImageData` object with the new dimensions, adjusted spacing, shifted origin, and processed scalars [S2, unofficial].
7.  **Mapper Configuration**: If using `f16`, set the mapper's `preferSizeOverAccuracy` flag to prioritize rendering speed over precision [S2, unofficial].
8.  **Binding**: Bind the data to the mapper for rendering [S2, unofficial].

#### Phase 2: Multi-Resolution Compression (VolCon/Compress)
This phase creates a hierarchical representation for efficient streaming and rendering [S1].

1.  **Stage 1 (VolCon)**: Compute the octree structure from input slice files. The tool `VolCon` reads the input data and stores it as octree brick files [S1].
2.  **Stage 2 (Compress)**: Calculate the wavelet representation based on the octree bricks generated in Stage 1. The tool `Compress` applies wavelet transform to the bricks [S1].

### Parameters
-   `AIR_HU`: Threshold value for air outside the body, used to compute the body bounding box. Default: -500 [S2, unofficial].
-   `PERF.crop`: Boolean flag enabling body bounding box cropping [S2, unofficial].
-   `PERF.down`: Integer downsampling factor applied to dimensions and spacing [S2, unofficial].
-   `PERF.tex`: String flag; when `'f16'`, triggers 16-bit float conversion and sets mapper size preference [S2, unofficial].
-   Octree depth and maximum level of wavelet compression: User-defined parameters controlling the granularity of the multi-resolution structure [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(raw_volume, labels=None, perf_config=None):
    """
    Preprocesses raw volume data for rendering.
    
    Args:
        raw_volume: 3D numpy array or similar structure of raw voxel values.
        labels: Optional 3D array of label IDs.
        perf_config: Dictionary containing PERF.crop, PERF.down, PERF.tex.
        
    Returns:
        vtkImageData object or processed volume data.
    """
    if perf_config is None:
        perf_config = {"crop": False, "down": 1, "tex": "f32"}

    # Step 1: Label Banding
    if labels is not None:
        # Apply bandLabeledHuValues to map labels to specific HU ranges
        volume = band_labeled_hu_values(raw_volume, labels)
    else:
        volume = raw_volume

    # Step 2: Cropping
    if perf_config.get("crop", False):
        # Compute body bounding box using AIR_HU threshold
        air_threshold = -500
        mask = volume > air_threshold
        if not np.any(mask):
            return None # Empty volume
        
        # Get bounding box coordinates
        coords = np.argwhere(mask)
        min_coords = coords.min(axis=0)
        max_coords = coords.max(axis=0)
        
        # Crop volume to bounding box
        volume = volume[min_coords[0]:max_coords[0]+1, 
                        min_coords[1]:max_coords[1]+1, 
                        min_coords[2]:max_coords[2]+1]
        
        # Shift origin to preserve world coordinates (logic depends on original origin)
        # Assuming original origin is (0,0,0) for simplicity, new origin is min_coords * spacing
        origin_shift = min_coords
    else:
        origin_shift = np.array([0, 0, 0])

    # Step 3: Downsampling
    down_factor = perf_config.get("down", 1)
    if down_factor > 1:
        # Stride downsampling
        volume = volume[::down_factor, ::down_factor, ::down_factor]

    # Step 4: Spacing Adjustment
    # Original spacing is assumed to be 1.0 or provided separately.
    # Here we adjust the logical spacing attribute.
    original_spacing = 1.0 # Placeholder for actual original spacing
    new_spacing = original_spacing * down_factor

    # Step 5: Format Conversion
    if perf_config.get("tex") == "f16":
        volume = volume.astype(np.float16)
        use_f16 = True
    else:
        volume = volume.astype(np.float32)
        use_f16 = False

    # Step 6: VTK Construction
    dimensions = volume.shape
    origin = origin_shift * original_spacing # Adjusted origin
    
    # Create VTK imageData (pseudo-code for VTK creation)
    vtk_data = create_vtk_image_data(
        dimensions=dimensions,
        spacing=new_spacing,
        origin=origin,
        scalars=volume
    )

    # Step 7: Mapper Configuration
    if use_f16:
        vtk_data.mapper.prefer_size_over_accuracy = True

    # Step 8: Binding
    # Bind data to mapper (handled by rendering engine)
    
    return vtk_data

def compress_volume_octree(slice_files, octree_depth, wavelet_levels):
    """
    Compresses volume data into an octree structure with wavelet compression.
    
    Args:
        slice_files: List of input slice files.
        octree_depth: Integer, depth of the octree.
        wavelet_levels: Integer, maximum level of wavelet compression.
        
    Returns:
        List of octree brick files.
    """
    # Stage 1: VolCon - Compute Octree Structure
    # VolCon reads slice files and creates octree bricks
    octree_bricks = volcon_compute_octree(slice_files, octree_depth)
    
    # Stage 2: Compress - Wavelet Representation
    # Compress applies wavelet transform to the bricks
    compressed_bricks = compress_wavelet(octree_bricks, wavelet_levels)
    
    return compressed_bricks
```

### Edge Cases and Pitfalls
-   **Precision**: Reinterpreting a `Uint8Array` buffer as `Float32Array` is safe in JavaScript engines because they are natively little-endian, provided metadata validates `byte_order === 'little'`. Using `DataView` for byte-by-byte reading is significantly slower (~15s+ for a 145MB volume) and can cause stale canvas frames during dataset switches [S2, unofficial].
-   **Empty Volumes**: If the cropping mask is empty (no voxels above `AIR_HU`), the pipeline should handle this gracefully [S2, unofficial].
-   **Resolution Mismatch**: If the resolution differs significantly between axes (e.g., Z vs XY), interpolation may be required to create a regular volume [S3, unofficial].

## Subtopics

### [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md)
The VolumeSegmenter Pipeline is an end-to-end method for compressing large RAW volumetric datasets into compact, spatially mapped bricks organized into selectable logical groups, enabling efficient rendering in WebGL2. It reduces stored and GPU-resident voxel data significantly, though it may increase rendering time due to sparse lookup and ray-marching overhead compared to monolithic source rendering. It differs from the [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by explicitly grouping bricks via intensity bands and adjacency rather than purely spatial partitioning, and builds on the general [Preprocessing Pipeline](6733-preprocessing-pipeline.md) framework.

### [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md)
The Volume Partitioning Pipeline is a data preprocessing method that splits a 3D volume into disjoint foreground regions, optionally partitions them spatially or by intensity, crops them to bounding boxes, downsamples, and exports them as raw files with metadata. It solves the problem of managing large 3D datasets by reducing memory footprint and enabling efficient out-of-core or chunked processing. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by focusing on intensity-based foreground extraction and spatial partitioning rather than semantic segmentation, and from [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md) by using a generic intensity-threshold approach rather than seismic-specific physics.

### [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md)
The 3D Seismic Data Compression Pipeline is a lossy compression method for 3D seismic data that processes volumes in 3D blocks (cubes), applying normalization, closed-loop differential pulse-code modulation (DPCM) with quantization, and Deflate compression to reduce file size while preserving trace integrity. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) and [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by operating on fixed 3D spatial-temporal blocks rather than semantic segments or arbitrary partitions, and from the legacy [**.sdc v2 Trace Compression Pipeline**](#-sdc-v2-trace-compression-pipeline) by using 3D predictive coding instead of 1D trace-level encoding.

### [Wild Scene Preprocessing & Test Set Selection](13103-wild-scene-preprocessing-test-set-selection.md)
A coarse pre-filtering pipeline to remove low-quality images dominated by transient objects from unconstrained photo collections, ensuring robust training and evaluation for view synthesis systems. It differs from sibling pipelines by focusing specifically on semantic quality (NIMA) and transient occlusion metrics (DeepLab) rather than geometric consistency or voxel-based segmentation.

### Slice Preprocessing
A preprocessing phase used for creating cropped slices to be distributed across the network for rendering. Each raw data slice is read once. Non-empty regions are segmented using a mask volume, cropped and written to disc memory. Out-of-core region growing is used if no mask has been provided. Concurrently, data extent information is consolidated into a series of slab-projection slices [S4, unofficial].

## Operating characteristics
-   **Time Complexity**: The preprocessing time depends on the octree depth and wavelet compression levels. Deeper octrees generate smaller brick files but increase file access overhead [S1].
-   **Memory Complexity**: The multi-resolution structure reduces the memory footprint by storing only relevant levels of detail. The size of brick files decreases with increased octree depth [S1].
-   **Performance**: Measured on a Core2Duo laptop with 2GB RAM, preprocessing time varies with dataset size and parameters [S1].
-   **Strengths**: Efficient rendering of large datasets through multi-resolution structures and cropping.
-   **Limitations**: Increased file access overhead with deeper octrees; potential increase in rendering time due to sparse lookup in segmented pipelines [S1].

## Background
The Preprocessing Pipeline originated in the context of gigascale multiresolution volume rendering on distributed display clusters [S1]. It was developed to handle large medical imaging datasets by creating efficient multi-resolution data structures. The pipeline has been adapted for various applications, including reinforcement learning for volume rendering transfer functions [S2, unofficial] and distributed volume visualization [S4, unofficial].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
[S2] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
[S3] 14 Acceleration Techniques for Volume Rendering Mark W. Jones 14 Acceleration Techniques for Volume Rendering Mark W. Jones, https://docslib.org/doc/3014834/14-acceleration-techniques-for-volume-rendering-mark-w-jones, fetched 2026-10-28 (unofficial)
[S4] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
