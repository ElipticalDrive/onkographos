# VolumeLoader

Also called: Volumetric Data Loader

## Overview
VolumeLoader is a generic volumetric data ingestion component that handles loading, optional resampling, and dtype conversion for medical imaging formats [S1, unofficial]. It solves the problem of converting heterogeneous raw volume data into a standardized, device-ready tensor format suitable for downstream rendering or optimization pipelines [S1, unofficial]. Unlike format-specific readers like [Raw Volume Data Reader](2259-raw-volume-data-reader.md) or [DICOM Volume Loading and Coordinate Reconstruction](2574-dicom-volume-loading-and-coordinate-reconstruction.md), VolumeLoader provides a unified interface with configurable storage precision and automatic overflow protection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A file path to a volumetric data file (supported formats include `.nii`, `.gz`, and `.npy`) [S1, unofficial].
*   **Output:** A device tensor with shape determined by `target_shape` or the original dimensions, and dtype determined by `storage_dtype` [S1, unofficial].

**Data Structures:**
The loader maintains internal state for configuration parameters but does not persist complex data structures during initialization [S1, unofficial]. The primary intermediate data structure is a NumPy array (for `.npy`) or nibabel image object (for `.nii`/`.gz`) before conversion to the final tensor [S1, unofficial].

**Parameters:**
*   `target_shape`: Optional tuple `(D, H, W)`. If `None`, the original dimensions are preserved [S1, unofficial].
*   `device`: The target device for the output tensor (e.g., `cpu`, `cuda`) [S1, unofficial].
*   `downscale_factor`: Integer factor $f > 1$. If provided, the volume is resampled to $(D//f, H//f, W//f)$ [S1, unofficial].
*   `storage_dtype`: String specifying the output precision. Must be one of `{'fp32', 'fp16', 'bf16'}` [S1, unofficial].
*   `enable_overflow_guard`: Boolean. If `True`, automatically resamples volumes exceeding ~16 million voxels to prevent downstream failures [S1, unofficial].

**Steps:**
1.  **Initialization:** Validate that `storage_dtype` is in `{'fp32', 'fp16', 'bf16'}` and raise `ValueError` if invalid [S1, unofficial].
2.  **File Detection and Loading:** Peek the raw range by loading `.nii`/`.gz` files with `nibabel` or `.npy` files with `numpy`, converting the result to a `float32` NumPy array [S1, unofficial].
3.  **Overflow Guard Check:** If `enable_overflow_guard` is `True`, check if the total number of voxels $V = D \times H \times W$ exceeds 16,000,000 [S1, unofficial]. If $V > 16,000,000$, calculate the minimal integer factor $f$ such that $V/f^3 \le 16,000,000$ and apply it [S1, unofficial].
4.  **Resampling:** If `target_shape` is provided, resample the volume to `target_shape` [S1, unofficial]. If `downscale_factor` is provided and $> 1$, resample the volume to $(D//f, H//f, W//f)$ [S1, unofficial]. If neither is provided, keep the original dimensions [S1, unofficial].
5.  **Normalization:** Apply min-subtraction normalization [S1, unofficial]. Let $v_{\min}$ be the minimum value in the volume and $v$ be a voxel value [S1, unofficial]. The normalized value $v'$ is computed as $v' = v - v_{\min}$ [S1, unofficial].
6.  **Device Transfer and Casting:** Convert the NumPy array to a tensor on the specified `device` and cast the tensor to `storage_dtype` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import nibabel as nib
import torch
import scipy.ndimage as ndi

def resample_volume(volume, new_shape):
    """Standard trilinear interpolation resampling [G]."""
    factors = tuple(np.array(new_shape) / np.array(volume.shape))
    return ndi.zoom(volume, factors, order=1)

def load_volume(file_path, target_shape=None, device='cpu', downscale_factor=None, storage_dtype='fp32', enable_overflow_guard=True):
    # 1. Validate storage_dtype
    valid_dtypes = {'fp32', 'fp16', 'bf16'}
    if storage_dtype not in valid_dtypes:
        raise ValueError(f"storage_dtype must be one of {valid_dtypes}, got {storage_dtype}")

    # 2. Load data based on extension
    if file_path.endswith('.nii') or file_path.endswith('.gz'):
        img = nib.load(file_path)
        volume = img.get_fdata().astype(np.float32)
    elif file_path.endswith('.npy'):
        volume = np.load(file_path).astype(np.float32)
    else:
        raise ValueError(f"Unsupported file format: {file_path}")

    original_shape = volume.shape
    D, H, W = original_shape
    total_voxels = D * H * W

    # 3. Overflow Guard
    if enable_overflow_guard and total_voxels > 16_000_000:
        import math
        required_factor = math.ceil((total_voxels / 16_000_000) ** (1/3))
        if downscale_factor is None:
            downscale_factor = required_factor
        else:
            downscale_factor = max(downscale_factor, required_factor)

    # 4. Resampling
    if target_shape is not None:
        volume = resample_volume(volume, target_shape)
    elif downscale_factor is not None and downscale_factor > 1:
        new_shape = (D // downscale_factor, H // downscale_factor, W // downscale_factor)
        volume = resample_volume(volume, new_shape)

    # 5. Normalization (Min-subtraction)
    v_min = np.min(volume)
    volume = volume - v_min

    # 6. Device Transfer and Casting
    tensor = torch.from_numpy(volume).to(device)
    
    if storage_dtype == 'fp32':
        tensor = tensor.float()
    elif storage_dtype == 'fp16':
        tensor = tensor.half()
    elif storage_dtype == 'bf16':
        tensor = tensor.bfloat16()
        
    return tensor
```

**Edge Cases and Pitfalls:**
*   **Unsupported Formats:** The loader explicitly supports `.nii`, `.gz`, and `.npy`; other formats raise a `ValueError` [S1, unofficial].
*   **Overflow:** Without `enable_overflow_guard`, very large volumes may cause memory errors in downstream operations that cannot handle >16M voxels [S1, unofficial].
*   **Normalization Scope:** The normalization is min-subtraction only and does not scale to $[0, 1]$, which may require downstream components to handle unbounded positive values if the original data had a large dynamic range above the minimum [S1, unofficial].

## Subtopics

### [Volume Data Loading and Normalization](2157-volume-data-loading-and-normalization.md)
Implements a C++ class for loading 3D and 4D volumetric data from `.npy` files, normalizing scalar values, and managing frame indices for time-varying volumes. It strictly relies on the NumPy binary format and applies a min-subtraction normalization strategy rather than standard min-max scaling to $[0, 1]$. It is chosen when the pipeline requires explicit frame index management for time-series data and strict NumPy compatibility.

### [Raw Volume Data Reader](2259-raw-volume-data-reader.md)
Ingests raw binary volume data (typically from CT scanners) into memory, applying domain-specific quantization to reduce precision, and preparing the data for GPU transfer. It solves the problem of converting high-precision raw sensor data into a format suitable for real-time raycasting or sparse voxel octree construction, often involving thresholding and bit-depth reduction. It differs from generic loaders by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline rather than treating data as opaque bytes.

### [DICOM Volume Loading and Coordinate Reconstruction](2574-dicom-volume-loading-and-coordinate-reconstruction.md)
Loads a 3D volumetric dataset from a series of DICOM image files, reconstructs the physical voxel grid by resolving coordinate system conventions, and converts raw pixel values into physical density. It specifically addresses the discrepancy between DICOM's pixel-center coordinate convention and the corner-point convention required by downstream volume rendering engines, while handling the non-cubic nature of medical voxels. It differs from generic volume loaders by explicitly managing DICOM-specific metadata (Rescale Slope/Intercept, Pixel Spacing, Slice Thickness) and orientation flips (rot90) necessary for correct anatomical alignment.

### [VTK Volume Loading & Normalization](2620-vtk-volume-loading-normalization.md)
Parses VTK Structured Points files, extracts voxel data, and normalizes scalar values to an 8-bit unsigned integer range $[0, 255]$. It converts arbitrary-precision medical or scientific volume data into a standardized format suitable for GPU-based raycasting or texture upload. It is distinguished from generic loaders by its specific reliance on the VTK Structured Points header format and its mandatory min-max normalization step, unlike [Raw Volume Data Reader](2259-raw-volume-data-reader.md) which assumes raw binary continuity, or [DICOM Volume Loading and Coordinate Reconstruction](2574-dicom-volume-loading-and-coordinate-reconstruction.md) which handles multi-frame DICOM series.

### [Volume Loading and Preprocessing](2623-volume-loading-and-preprocessing.md)
Handles the ingestion, coordinate alignment, and intensity normalization of volumetric data, specifically targeting medical imaging formats like CT scans. It solves the problem of converting raw, device-specific metadata (such as IJK-to-RAS affine transforms) into a standardized, GPU-ready tensor format. It differs from [Raw Volume Data Reader](2259-raw-volume-data-reader.md) by handling structured medical metadata and from [DICOM Volume Loading and Coordinate Reconstruction](2574-dicom-volume-loading-and-coordinate-reconstruction.md) by focusing on the post-loading transform application and intensity scaling rather than the initial file parsing and geometry extraction.

### [SOG Volume Reader](3580-sog-volume-reader.md)
The SOG Volume Reader is a recursive algorithm that loads a hierarchical volumetric format (SOG) into a dense 3D grid by parsing a binary header and traversing internal, empty, leaf, and pseudo-leaf node types. It differs from flat raw readers by reconstructing the full voxel array from a compressed octree-like structure, enabling efficient storage of sparse or block-structured medical imaging data.

### AVS FLD Volume Loader
Parses legacy AVS FLD format to extract 3D scalar volume data and dimensions [S2, unofficial]. It reads the first 16 bytes to verify the magic string "# AVS field file", scans for the `\f` byte to determine header length, and decodes header key-value pairs using regex splitting [S2, unofficial]. It validates that `ndim == 3`, `veclen == 1`, and `data == "uniform"`, then maps data types ("byte" to 1, "short" to 2) and unpacks the data using Fortran column-major order [S2, unofficial]. Pitfall: Only data types 1 (byte) and 2 (short) are supported; other types raise an exception [S2, unofficial].

### DICOM Node
A node that loads DICOM data, handles mesh escalation to match stored scale, interpolates voxel values, and applies jittering to improve quality during ray casting with few steps [S3, unofficial]. It escalates the mesh according to DICOM data specifications, considers three different types of data based on storage byte count, interpolates values between nearest voxels, and applies jittering (varying ray start distances) to improve quality in low-step scenarios [S3, unofficial]. Parameter: Node color is Grey if empty, Orange if a DICOM file is uploaded [S3, unofficial].

### NeXus to NRRD/NPZ Conversion
A method to load volume data from a NeXus HDF5 file, handling NXdata groups and axis attributes, and save it as NRRD or NPZ format with normalized float32 data and spatial metadata [S4, unofficial]. It opens the NeXus file using `h5py`, loads the volume from the specified group (default "processed/reciprocal_space") and data name (default "volume"), and extracts spacings and origin for NRRD headers [S4, unofficial]. Pitfall: If $v_{max}$ equals $v_{min}$ during normalization, division by zero occurs, potentially resulting in NaN values [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Time complexity is dominated by file I/O and resampling operations [S1, unofficial]. Memory complexity scales with the loaded volume size in `float32` or `storage_dtype` during processing [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Scales with input resolution; `downscale_factor` and `enable_overflow_guard` reduce effective resolution to fit ~16M voxel constraints [S1, unofficial].

**Strengths and Limitations:**
Strengths include unified format support, overflow protection, and configurable precision [S1, unofficial]. Limitations include lack of native DICOM/VTK/AVS parsing and unbounded min-subtraction normalization [S1, unofficial].

**When to use it and when not:**
Use for NIfTI or NumPy inputs requiring simple loading with overflow protection [S1, unofficial]. For formats requiring specific metadata handling (e.g., DICOM, VTK, AVS), use the specialized subtopics instead [S2, S3, S4, unofficial].

## Background
VolumeLoader originates from the `vgs` project by Valentin Kraft, an unofficial repository focused on volumetric graphics and optimization [S1, unofficial]. It was developed to provide a standardized data ingestion layer for training and rendering pipelines that handle medical imaging data, addressing the variability in file formats and precision requirements [S1, unofficial].

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S3] Node-Based Shader Editor for Volume Rendering in WebGL, https://miccai-sb.github.io/materials/mec2020/2020_UbietoV.pdf, fetched 2026-10-28 (unofficial)
[S4] PeterC-DLS/webgl_volume_demo, https://github.com/PeterC-DLS/webgl_volume_demo, fetched 2026-10-27 (unofficial)
[S5] caitlin-leonard/medviewer, https://github.com/caitlin-leonard/medviewer, fetched 2026-10-27 (unofficial)
