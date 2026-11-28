# DICOM Rescale (HU Conversion)
Also called: Gray Value Calibration, Intensity Rescaling, Pixel Value Rescaling, RescaleIntercept, RescaleSlope

## Overview
Converts raw DICOM pixel values to standardized physical units, typically Hounsfield Units (HU), using affine parameters stored in the DICOM header [S1, unofficial]. It differs from [Volume Normalization](1823-volume-normalization.md) by preserving absolute physical meaning rather than mapping to a generic $[0, 1]$ range, and from [Dynamic Range Synchronization](22222-dynamic-range-synchronization.md) by operating per-voxel based on metadata rather than global histogram statistics.

## How it works
**Inputs:**
- Raw voxel data array $V_{raw}$ of size $N$.
- Rescale Slope ($m$) and Rescale Intercept ($b$) from DICOM tags [S1, unofficial].

**Outputs:**
- Physical unit voxel data array $V_{phys}$ of size $N$.

**Data Structures:**
- `nSlope` (short): DICOM tag (0x0028,0x1053). Typical value: 1 [S1, unofficial].
- `nIntercept` (short): DICOM tag (0x0028,0x1052). Typical value: 0 [S1, unofficial].
- `pData`: Pointer to the raw voxel data buffer [S1, unofficial].

**Steps:**
1. Validate input pointer. If `pData` is NULL, return immediately [S1, unofficial].
2. Iterate through each voxel index $i$ from $0$ to $N-1$.
3. Apply the affine transformation:
   $$ V_{phys}[i] = (V_{raw}[i] + b) \times m $$
   where $b$ is `nIntercept` and $m$ is `nSlope` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rescale_hu(p_data: list[int], n_size: int, n_intercept: int, n_slope: int) -> list[float]:
    """
    Converts raw DICOM pixel values to Hounsfield Units.
    
    Args:
        p_data: List of raw integer pixel values.
        n_size: Number of voxels.
        n_intercept: Rescale Intercept (DICOM 0x0028,0x1052).
        n_slope: Rescale Slope (DICOM 0x0028,0x1053).
        
    Returns:
        List of float values in HU.
    """
    if p_data is None:
        return []
    
    result = [0.0] * n_size
    for i in range(n_size):
        # Formula: (Raw + Intercept) * Slope
        result[i] = (p_data[i] + n_intercept) * n_slope
        
    return result
```

**Parameters:**
- `nIntercept`: Offset added to raw voxel values. Stored in `m_nRescaleIntercept` [S1, unofficial].
- `nSlope`: Multiplier applied after adding the intercept. Stored in `m_nRescaleSlope` [S1, unofficial].

**Edge Cases and Pitfalls:**
- Null pointer check is required before processing; if `pData` is NULL, the function returns immediately without modifying data [S1, unofficial].
- Precision: The result is typically stored as floating point to accommodate fractional HU values, though the inputs are integers [S1, unofficial].

## Subtopics

### DICOM Preprocessing
A specific pipeline variant that includes HU conversion as a step [S2, unofficial].
- **What it is:** A sequence of operations preparing DICOM slices for training or rendering [S2, unofficial].
- **Details:**
  1. Sort DICOM slices by `ImagePositionPatient` Z-coordinate [S2, unofficial].
  2. Convert HU using per-slice `RescaleSlope` and `RescaleIntercept` tags [S2, unofficial].
  3. Clip HU values to $[-1000, 400]$ [S2, unofficial].
  4. Normalize to $[0, 1]$ using:
     $$ v_{norm} = \frac{HU - (-1000)}{400 - (-1000)} $$
     [S2, unofficial]
- **When to choose:** When preparing data for neural network training (e.g., TensoRF) where normalized inputs and anatomical clipping are required [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels, as it performs a single pass over the data [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the output buffer, or $O(1)$ if performed in-place [S1, unofficial].
- **Strengths:** Preserves physical accuracy required for medical diagnosis and quantitative analysis [S1, unofficial].
- **Limitations:** Requires valid DICOM metadata; if tags are missing, default values (slope=1, intercept=0) may be assumed, leading to incorrect physical units [G].
- **When to use:** When the downstream application requires physical density information (e.g., tissue classification based on HU) [S1, unofficial].
- **When not to use:** When only relative intensity contrast is needed and physical units are irrelevant [G].

## Background
Originated from the DICOM standard (Digital Imaging and Communications in Medicine) to ensure consistent interpretation of pixel data across different modalities and manufacturers [S1, unofficial]. The `RescaleSlope` and `RescaleIntercept` tags allow vendors to store data in compressed or scaled integer formats while preserving the original physical calibration [S1, unofficial]. Implemented in various medical imaging libraries and research pipelines, such as the IGS project [S1, unofficial] and lung nodule volumetry research [S2, unofficial].

## Sources
[S1] TechCloud-x/IGS, https://github.com/TechCloud-x/IGS, fetched 2026-10-26 (unofficial)
[S2] Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF, https://doi.org/10.48550/arxiv.2606.02639, fetched 2026-10-26 (unofficial)
