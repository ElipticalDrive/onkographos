# VolumeLoader

Also called: Volumetric Data Loader
## Overview
VolumeLoader is a generic volumetric data ingestion method that handles loading, optional resampling, and coordinate system alignment for various medical imaging formats [S1, unofficial]. It solves the problem of converting heterogeneous source files (e.g., NIfTI, NumPy) into a standardized, device-ready tensor while managing memory constraints through automatic downscaling [S1, unofficial]. It differs from specialized loaders like the Raw Volume Data Reader or DICOM-specific loaders by providing a unified interface with built-in overflow guards and dtype validation, rather than focusing on specific file headers or physical coordinate reconstruction [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: File path(s) to volumetric data (e.g., `.nii`, `.gz`, `.npy`).
- Output: A tensor on the specified `device` with shape `target_shape` (or original shape) and dtype `storage_dtype`.

**Data Structures:**
- Configuration object containing:
  - `target_shape`: Optional tuple `(D, H, W)` for resampling.
  - `device`: Target compute device (e.g., CPU, GPU).
  - `downscale_factor`: Integer factor for resolution reduction.
  - `storage_dtype`: One of `{'fp32', 'fp16', 'bf16'}`.
  - `enable_overflow_guard`: Boolean flag for automatic large-volume handling.

**Steps:**
1. **Initialization:** Create a `VolumeLoader` instance with the configuration parameters above [S1, unofficial].
2. **Validation:** Verify that `storage_dtype` is in `{'fp32', 'fp16', 'bf16'}`. Raise `ValueError` if not [S1, unofficial].
3. **Raw Range Peeking:** Load the raw data into a temporary float32 numpy array to determine the data range and dimensions:
   - For `.nii` or `.gz` files: Use `nibabel` to load and convert to `float32` numpy array [S1, unofficial].
   - For `.npy` files: Use `numpy.load` [S1, unofficial].
4. **Overflow Guard Check:** If `enable_overflow_guard` is True, check if the total voxel count exceeds ~16 million. If it does, automatically apply `downscale_factor` to reduce dimensions to `(D//factor, H//factor, W//factor)` [S1, unofficial].
5. **Resampling:** If `target_shape` is provided, or if `downscale_factor` > 1, resample the volume to the target dimensions. If `target_shape` is None, keep original dimensions (subject to overflow guard) [S1, unofficial].
6. **Device Transfer:** Cast the data to `storage_dtype` and transfer to the specified `device` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import nibabel as nib
from typing import Optional, Tuple, Union

def load_volume(
    filepath: str,
    target_shape: Optional[Tuple[int, int, int]] = None,
    device: str = "cpu",
    downscale_factor: int = 1,
    storage_dtype: str = "fp32",
    enable_overflow_guard: bool = True
) -> np.ndarray:
    # 1. Validate storage_dtype
    valid_dtypes = {"fp32", "fp16", "bf16"}
    if storage_dtype not in valid_dtypes:
        raise ValueError(f"storage_dtype must be one of {valid_dtypes}, got {storage_dtype}")

    # 2. Peek raw range and load into float32 numpy array
    if filepath.endswith(('.nii', '.gz')):
        # Load using nibabel
        nib_img = nib.load(filepath)
        raw_data = nib_img.get_fdata().astype(np.float32)
    elif filepath.endswith('.npy'):
        # Load using numpy
        raw_data = np.load(filepath).astype(np.float32)
    else:
        raise ValueError(f"Unsupported file format: {filepath}")

    original_shape = raw_data.shape
    total_voxels = np.prod(original_shape)

    # 3. Overflow Guard Check
    if enable_overflow_guard and total_voxels > 16_000_000:
        if downscale_factor <= 1:
            downscale_factor = 2  # Default downscale if not specified but overflow occurs
        # Apply downscale factor
        new_shape = tuple(dim // downscale_factor for dim in original_shape)
        # Note: Simple integer division may truncate; resampling logic would typically use interpolation
        # For this loader, we assume the downscale_factor is applied via resampling
        raw_data = resample_volume(raw_data, new_shape)
        original_shape = new_shape

    # 4. Resampling to target_shape
    if target_shape is not None:
        raw_data = resample_volume(raw_data, target_shape)
    elif downscale_factor > 1:
        new_shape = tuple(dim // downscale_factor for dim in original_shape)
        raw_data = resample_volume(raw_data, new_shape)

    # 5. Cast and Transfer
    dtype_map = {
        "fp32": np.float32,
        "fp16": np.float16,
        "bf16": np.float16  # Note: bf16 is not natively supported in standard numpy, handled by device-specific logic or casting to fp16 as approximation if no native support [G]
    }
    np_dtype = dtype_map[storage_dtype]
    result = raw_data.astype(np_dtype)

    # Transfer to device (placeholder for actual device transfer logic)
    # In a real implementation, this would move data to GPU memory
    return result

def resample_volume(data: np.ndarray, target_shape: Tuple[int, int, int]) -> np.ndarray:
    """
    Simple resampling using interpolation.
    In a real implementation, this would use scipy.ndimage.zoom or similar.
    """
    from scipy.ndimage import zoom
    zoom_factors = [t / o for t, o in zip(target_shape, data.shape)]
    return zoom(data, zoom_factors, order=1) # Linear interpolation
```

**Parameters:**
- `target_shape`: Optional tuple `(D, H, W)`. If `None`, original dimensions are kept [S1, unofficial].
- `downscale_factor`: Integer. When > 1, volumes are resampled to `(D//factor, H//factor, W//factor)` [S1, unofficial].
- `storage_dtype`: One of `{'fp32', 'fp16', 'bf16'}` [S1, unofficial].
- `enable_overflow_guard`: Boolean. Automatically resamples very large volumes to avoid downstream operations failing on >~16M voxels [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** `bf16` is not natively supported in standard NumPy; implementation may require casting to `fp16` or using device-specific tensors [G].
- **Boundaries:** Integer division in `downscale_factor` may truncate dimensions, potentially losing data if dimensions are not perfectly divisible by the factor [G].
- **Degenerate Input:** Files with unsupported formats raise `ValueError` [S1, unofficial].

## Subtopics

### [Volume Data Loading and Normalization](2157-volume-data-loading-and-normalization.md)
Implements a C++ class for loading 3D/4D volumetric data from `.npy` files, normalizing scalar values, and managing frame indices for time-varying volumes. It differs from siblings by strictly relying on the NumPy binary format and applying a min-subtraction normalization strategy rather than standard min-max scaling to $[0, 1]$. Choose this when ingesting pre-saved NumPy arrays with consistent data types and value ranges.

### [Raw Volume Data Reader](2259-raw-volume-data-reader.md)
Specialized loader for ingesting raw binary volume data files directly into memory, solving the conversion of high-precision raw sensor data (e.g., CT scanners) into a usable format. It differs by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline rather than treating data as opaque bytes. Choose this for raw binary continuity where quantization is required at load time.

### [DICOM Volume Loading and Coordinate Reconstruction](2574-dicom-volume-loading-and-coordinate-reconstruction.md)
Loads 3D volumetric datasets from DICOM image series, reconstructs the physical voxel grid, and converts raw pixel values into physical density. It specifically addresses the discrepancy between DICOM's pixel-center coordinate convention and the corner-point convention required by downstream engines, while handling non-cubic voxels. It differs by explicitly managing DICOM metadata (Rescale Slope/Intercept, Pixel Spacing, Slice Thickness) and orientation flips. Choose this for medical imaging data requiring anatomical alignment.

### [VTK Volume Loading & Normalization](2620-vtk-volume-loading-normalization.md)
Parses VTK Structured Points files, extracts voxel data, and normalizes scalar values to an 8-bit unsigned integer range $[0, 255]$. It solves the problem of converting arbitrary-precision medical/scientific data into a standardized format for GPU-based raycasting. It is distinguished by its specific reliance on the VTK Structured Points header format and mandatory min-max normalization, unlike the Raw Volume Data Reader or DICOM loader. Choose this for VTK-formatted data requiring 8-bit normalization.

### [Volume Loading and Preprocessing](2623-volume-loading-and-preprocessing.md)
Handles ingestion from various medical imaging formats (DICOM, raw binary) and performs normalization, coordinate system alignment, and memory layout preparation. It distinguishes itself by focusing on the complete pipeline from file I/O to GPU-ready tensor preparation, including affine matrix transformations (RAS to IJK) and Hounsfield Unit scaling. Unlike the Raw Volume Data Reader or DICOM loader, it prepares the final 3D volume tensor for direct rendering. Choose this for end-to-end preprocessing of medical data.

### [SOG Volume Reader](3580-sog-volume-reader.md)
Recursive algorithm that loads a hierarchical volumetric format (SOG) into a dense 3D grid, utilizing internal, empty, leaf, and pseudo-leaf node types. It differs from standard raw readers by parsing a binary header and recursively traversing an octree structure to populate the output array, allowing for sparse/compressed representations. Choose this for SOG-formatted hierarchical data.

### AVS FLD Volume Loader
Parses legacy AVS FLD format to extract 3D scalar volume data and dimensions [S2, unofficial]. It reads the first 16 bytes to verify the magic string "# AVS field file", scans for the `\f` byte to determine header length, and decodes header fields (ndim, dim1, dim2, dim3, etc.) using regex splitting [S2, unofficial]. It validates `ndim==3`, `veclen==1`, and `data=="uniform"`, then reads `dim_x * dim_y * dim_z * data_type` bytes, unpacking as 'B' (byte) or 'H' (short) and reshaping to `(dim_x, dim_y, dim_z)` with Fortran order [S2, unofficial]. Pitfall: Only data types 1 (byte) and 2 (short) are supported; others raise an exception [S2, unofficial]. Choose this for legacy AVS FLD data.

## Operating characteristics
- **Time Complexity:** Dominated by file I/O and resampling. Resampling is $O(N)$ where $N$ is the number of voxels in the output volume.
- **Memory Complexity:** $O(N)$ for the loaded tensor. The overflow guard limits peak memory usage by reducing $N$ for very large inputs.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with resolution via `downscale_factor` and `target_shape`.
- **Strengths:** Unified interface for multiple formats, built-in overflow protection, flexible dtype support.
- **Limitations:** Limited to specific formats (NIfTI, NPY); does not handle complex DICOM coordinate systems or raw binary quantization natively.
- **When to use:** When loading standard volumetric formats (NIfTI, NumPy) for deep learning or rendering pipelines where memory constraints are a concern.
- **When not:** When specific medical coordinate reconstruction (DICOM) or raw binary quantization is required.

## Background
Originated in the VGS (ValentinKraft/vgs) project as a core component for handling volumetric data in medical imaging and rendering contexts [S1, unofficial]. It addresses the need for a robust, memory-aware loader in modern volumetric rendering pipelines that often involve large datasets and GPU acceleration.

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
