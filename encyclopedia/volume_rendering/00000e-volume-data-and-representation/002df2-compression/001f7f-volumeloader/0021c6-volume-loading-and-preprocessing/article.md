# Volume Loading and Preprocessing

Also called: CT Volume Loading

## Overview
This method loads volumetric data from medical imaging formats, applies optional resampling, and aligns coordinate systems to a standardized space [S1, unofficial]. It differs from [DICOM Volume Loading and Coordinate Reconstruction](2574-dicom-volume-loading-and-coordinate-reconstruction.md) by focusing on post-loading affine transformation application and intensity scaling rather than initial file parsing and geometry extraction [S1, unofficial].

## How it works
**Inputs:** Raw volumetric data, metadata containing affine transformation matrices, target voxel spacing, and intensity scaling bounds [S1, unofficial].
**Outputs:** A normalized, channel-first tensor on the target device (e.g., CUDA) and the inverse affine matrix for coordinate mapping [S1, unofficial].

**Data Structures:**
*   **Affine Matrix:** A $4 \times 4$ matrix representing the linear transformation between coordinate spaces (e.g., IJK to RAS) [S1, unofficial].
*   **Tensor:** A multi-dimensional array with shape $(C, D, H, W)$ or $(C, Z, Y, X)$ containing floating-point values [S1, unofficial].

**Steps:**
1.  **Load Image:** Read the raw voxel data and associated metadata using a library such as MONAI's `LoadImage` [S1, unofficial].
2.  **Ensure Channel First:** Reshape the data to ensure the channel dimension is first using `EnsureChannelFirst` [S1, unofficial].
3.  **Resample (Spacing):** If the source voxel spacing differs from the target spacing, resample the volume using the `Spacing` transform [S1, unofficial].
4.  **Intensity Scaling:** Apply `ScaleIntensityRange` to map raw intensity values to a normalized range, clipping values outside the source bounds [S1, unofficial].
5.  **Extract Affine:** Retrieve the IJK-to-RAS affine matrix from the volume metadata [S1, unofficial].
6.  **Compute Inverse Affine:** Calculate the inverse of the RAS-to-IJK matrix to facilitate coordinate mapping during rendering [S1, unofficial].
7.  **Batch Dimension:** Expand the tensor to add batch and channel dimensions using `expand(1, 1, -1, -1, -1)` [S1, unofficial].
8.  **Device Transfer:** Move the tensor and affine matrix to the target hardware (e.g., CUDA) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import monai.transforms as mt

def load_and_preprocess_volume(file_path, target_spacing_mm, intensity_min, intensity_max):
    # 1. Load image and metadata
    loader = mt.LoadImage()
    image, meta = loader(file_path)
    
    # 2. Ensure channel first
    ensure_ch = mt.EnsureChannelFirst()
    image = ensure_ch(image)
    
    # 3. Resample to target spacing
    spacing_transform = mt.Spacing(new_spacing=target_spacing_mm)
    image = spacing_transform(image)
    
    # 4. Scale intensity to [0, 1] with clipping
    scale_transform = mt.ScaleIntensityRange(
        minval=intensity_min,
        maxval=intensity_max,
        out_min=0.0,
        out_max=1.0
    )
    image = scale_transform(image)
    
    # 5. Extract IJK to RAS affine from metadata
    ijk_to_ras_affine = meta['affine']
    
    # 6. Compute RAS to IJK inverse affine
    ras_to_ijk_affine = torch.inverse(torch.tensor(ijk_to_ras_affine, dtype=torch.float32))
    
    # 7. Add batch and channel dimensions using expand
    image = image.expand(1, 1, -1, -1, -1)
    
    # 8. Move to CUDA
    if torch.cuda.is_available():
        image = image.to('cuda')
        ras_to_ijk_affine = ras_to_ijk_affine.to('cuda')
        
    return image, ras_to_ijk_affine
```

**Parameters:**
*   `target_spacing_mm`: List of floats representing voxel dimensions in millimeters (e.g., `[2.5, 2.5, 3.0]`) [S1, unofficial].
*   `intensity_min`, `intensity_max`: Floats defining the clipping range for intensity scaling (e.g., `-3024`, `3024` for CT) [S1, unofficial].

**Edge cases and pitfalls:**
*   Intensity values outside `[intensity_min, intensity_max]` are clipped to `[0, 1]` rather than extrapolated [S1, unofficial].
*   The affine matrix must be converted to a tensor with explicit `dtype=torch.float32` before inversion to avoid precision loss on GPU [S1, unofficial].

## Subtopics

### Brick Density Volume Loading
This variant loads volumetric data into a `BrickGrid` structure, supporting both CT and Ocean Data formats [S2, unofficial]. It differs from standard loading by iterating over the Y-axis for CT data and parsing SciData files for Ocean Data, normalizing spatial locations and O2 concentration to $[0, 1]$ [S2, unofficial]. Choose this variant when working with oceanographic or specialized CT datasets requiring brick-based memory management [S2, unofficial]. Index bounds checking must be enforced via the `set()` method to prevent out-of-bounds writes to `m_brickData` [S2, unofficial].

### Brick Loading Pre-processing
This method applies pre-processing in a brick-loading thread to convert high-precision data (e.g., 16-bit) to unsigned 8-bit integers for efficient transfer [S3, unofficial]. It differs from full-volume preprocessing by operating on individual bricks and supports `scale`, `scaleToMax`, and `logTransform` methods [S3, unofficial]. Choose this variant when bandwidth optimization is critical and brick-level granularity is available [S3, unofficial]. Values outside `[min * 255, max * 255]` are zeroed, and bricks containing only zeros transmit a special `EMPTY` value instead of data [S3, unofficial].

### Volume Data Provider Contract and Fallback Pipeline
This variant defines the data-provider contract and fallback pipeline for the Volume3D render paths [S4]. It differs from direct loading by enforcing explicit provider payload mapping, resolver/registry wiring, and tests for new surface `renderMode` configurations [S4]. Choose this variant when integrating with the Cornerstone3D ecosystem or when routing between WebGPU and CPU paths is required [S4]. The `DefaultVolume3DDataProvider.load` contract accepts only `vtkVolume3d` and `vtkGeometry3d`, routing non-volume branches to geometry payloads and following a data path of full resolution → reduced level → slab or bricks → refuse [S4].

## Operating characteristics
Not given in the sources.

## Background
This workflow utilizes MONAI transforms for medical image preprocessing [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S3] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S4] docs(volume3d): analysis of the three Volume3D render paths and an improvement plan, https://github.com/cornerstonejs/cornerstone3D/pull/2853, fetched 2026-10-26
