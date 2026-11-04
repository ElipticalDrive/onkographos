# DICOM CT Projection Loading and Geometry Extraction

Also called: CT Projection Geometry Extraction & Coordinate Conversion

## Overview
This method loads raw CT projection data from a directory of DICOM files, extracts scanner geometry parameters, applies Hounsfield Unit (HU) calibration, and constructs a ConeBeamGeometry object suitable for tomographic reconstruction [S1, unofficial]. It specifically addresses the challenge of interpreting non-standard or vendor-specific DICOM metadata (such as angle definitions and flying focal spot offsets) to correctly map 2D detector coordinates to 3D ray paths [S1, unofficial]. It differs from standard CT loading by explicitly handling cylindrical-to-rectangular coordinate conversion for the ray transform and correcting for pixel-center vs. pixel-corner detector geometry [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A directory path containing `.dcm` files.
- `indices`: Optional advanced indexing (slice or list) to load specific projection angles [S1, unofficial].
- `interp`: Interpolation method string (default `'linear'`) used for the ray transform [S1, unofficial].

**Outputs:**
- `data_array`: A 3D NumPy array of shape `[num_projections, detector_columns, detector_rows]` with `dtype=float32`, containing calibrated projection values [S1, unofficial].
- `angles`: A 1D NumPy array of shape `[num_projections]` containing the corrected source angles in radians [S1, unofficial].
- `geometry`: A `ConeBeamGeometry` object (from `odl.tomo`) containing the source, detector, and ray configuration [S1, unofficial].

### Data Structures
1.  **DICOM Dataset Object**: Standard pydicom Dataset containing tags such as `PixelData`, `RescaleSlope`, `RescaleIntercept`, `HUCalibrationFactor`, `DetectorFocalCenterRadialDistance`, `DetectorCentralElement`, `DetectorSpacing`, and angular position tags.
2.  **Projection Array**: `float32` array, shape `[N, C, R]` where $N$ is the number of projections, $C$ is detector columns, $R$ is detector rows.
3.  **Geometry Object**: Contains `src_radius`, `det_radius`, `detector_shape`, `detector_spacing`, `angles`, and `detector_bounds`.

### Steps

1.  **File Discovery and Sorting**: List all `.dcm` files in the input directory. Sort them to ensure consistent angular ordering [S1, unofficial]. If `indices` are provided, filter the file list accordingly [S1, unofficial].

2.  **Metadata Extraction**: For each DICOM file, extract the following tags:
    -   Detector dimensions: `DetectorRows`, `DetectorCols` (or inferred from `PixelData` shape).
    -   Calibration: `RescaleSlope` ($m$), `RescaleIntercept` ($b$), `HUCalibrationFactor` ($H$) [S1, unofficial].
    -   Geometry: `DetectorSpacing` (pitch/spacing), `DetectorFocalCenterRadialDistance` (source-to-detector center distance component), `DetectorCentralElement` (center offset) [S1, unofficial].
    -   Angular Position: The raw angle tag provided by the scanner [S1, unofficial].

3.  **Data Loading and Reshaping**:
    -   Read `PixelData` as `uint16`.
    -   Reshape to `[rows, cols]` in Fortran order (column-major).
    -   Transpose the array to `[cols, rows]` to align with standard image coordinate systems where the first axis is horizontal (u) and the second is vertical (v) [S1, unofficial].

4.  **HU Calibration**:
    Apply the linear calibration to convert raw pixel values to Hounsfield Units. For each projection $P_{raw}$:
    $$ P = \frac{(P_{raw} \cdot m + b)}{H} $$
    where $m$ is `RescaleSlope`, $b$ is `RescaleIntercept`, and $H$ is `HUCalibrationFactor` [S1, unofficial]. Store the result in `data_array` [S1, unofficial].

5.  **Angle Extraction and Correction**:
    -   Extract raw angles from the DICOM metadata.
    -   Apply unwrapping to handle discontinuities at $\pm \pi$: `angles = np.unwrap(angles)`.
    -   Apply a global offset correction: `angles = -angles - np.pi` [S1, unofficial]. This corrects for the specific angle definition used by the scanner [S1, unofficial].

6.  **Detector Geometry Calculation**:
    -   Compute pixel size from `DetectorSpacing` [S1, unofficial].
    -   Calculate detector bounds in physical units. The method corrects for the difference between pixel centers and corners. Let $S$ be the detector shape `[cols, rows]` and $P$ be the pixel size. Let $C_{center}$ be `DetectorCentralElement`.
    -   Minimum corner ($minp$):
        $$ minp = -(C_{center} - 0.5) \cdot P $$
    -   Maximum corner ($maxp$):
        $$ maxp = minp + S \cdot P $$
    -   These bounds define the physical extent of the detector plane [S1, unofficial].

7.  **Source and Detector Radii**:
    -   Extract `src_radius` from `DetectorFocalCenterRadialDistance` [S1, unofficial].
    -   Compute `det_radius` using a constant radial distance parameter (often derived from scanner specs or metadata if available, otherwise assumed constant) minus `src_radius`:
        $$ det\_radius = ConstantRadialDistance - src\_radius $$
    -   The total source-to-detector distance is $d = src\_radius + det\_radius$ [S1, unofficial].

8.  **Pitch Calculation (if missing)**:
    If axial position metadata is present but pitch is not explicitly defined, compute it manually:
    $$ pitch = \frac{\Delta axial\_pos}{(\theta_{max} - \theta_{min}) / (2\pi)} $$
    where $\Delta axial\_pos$ is the total axial travel and $\theta_{max/min}$ are the min/max angles [S1, unofficial].

9.  **Flying Focal Spot Correction**:
    -   Extract source shifts for partial flying focal spot corrections [S1, unofficial].
    -   Apply mean offsets to the angles and detector bounds as first-order corrections. Note that this is a simplified handling; full dynamic correction may require more complex modeling [S1, unofficial].

10. **Coordinate Conversion for Ray Transform**:
    The ray transform (e.g., in ODL/TIGRE) often expects rectangular coordinates $(u, v)$, but the geometry is defined in cylindrical coordinates relative to the rotation axis $(up, vp)$. Convert the detector coordinates:
    $$ u = d \cdot \arctan\left(\frac{up}{d}\right) $$
    $$ v = \frac{d}{\sqrt{d^2 + up^2}} \cdot vp $$
    where $d = src\_radius + det\_radius$, $up$ is the vertical detector coordinate, and $vp$ is the horizontal detector coordinate [S1, unofficial].

11. **Geometry Assembly**:
    Create an `odl.tomo.ConeBeamGeometry` object using the computed `src_radius`, `det_radius`, `detector_shape`, `detector_spacing`, `detector_bounds`, and `angles` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import pydicom
import os
from odl.tomo import ConeBeamGeometry

def load_dicom_projections_and_geometry(folder_path, indices=None):
    """
    Load CT projections from DICOM files and extract geometry.
    """
    # 1. List and sort files
    dcm_files = sorted([f for f in os.listdir(folder_path) if f.endswith('.dcm')])
    if indices is not None:
        dcm_files = [dcm_files[i] for i in indices]
    
    num_projections = len(dcm_files)
    if num_projections == 0:
        raise ValueError("No DICOM files found.")

    # 2. Read first file to determine shape and extract static geometry
    ds0 = pydicom.dcmread(os.path.join(folder_path, dcm_files[0]))
    pixel_data = ds0.PixelData
    # Reshape to [rows, cols] in Fortran order, then transpose to [cols, rows]
    rows = ds0.Rows
    cols = ds0.Columns
    proj_raw = np.frombuffer(pixel_data, dtype=np.uint16).reshape((rows, cols), order='F').T
    
    # Extract calibration factors
    m = getattr(ds0, 'RescaleSlope', 1.0)
    b = getattr(ds0, 'RescaleIntercept', 0.0)
    H = getattr(ds0, 'HUCalibrationFactor', 1.0)
    
    # Extract geometry parameters
    src_radius = ds0.DetectorFocalCenterRadialDistance
    # Assuming ConstantRadialDistance is a known constant or derived from metadata
    # In many implementations, this is a fixed scanner parameter.
    # If not available, it must be provided externally. Here we assume it's available.
    # Note: The source notes imply 'ConstantRadialDistance' is used.
    # We will assume a variable 'constant_radial_distance' is passed or known.
    # For this implementation, we assume it is a global constant or extracted similarly.
    # Let's assume it's extracted from the first dataset if present, otherwise a constant.
    if hasattr(ds0, 'ConstantRadialDistance'):
        constant_radial_distance = ds0.ConstantRadialDistance
    else:
        # [G] Fallback: If not in metadata, this value is scanner-specific and must be provided.
        # Common values might be derived from Source-to-Isocenter + Isocenter-to-Detector.
        constant_radial_distance = src_radius + 540.0 # Example placeholder, not from source.
        
    det_radius = constant_radial_distance - src_radius
    d = src_radius + det_radius
    
    # Detector spacing and central element
    det_spacing = ds0.DetectorSpacing # Usually a tuple or scalar
    if isinstance(det_spacing, (list, tuple)):
        pixel_size_u, pixel_size_v = det_spacing
    else:
        pixel_size_u = pixel_size_v = det_spacing
        
    center_element = ds0.DetectorCentralElement
    
    # 3. Initialize arrays
    data_array = np.zeros((num_projections, cols, rows), dtype=np.float32)
    angles = np.zeros(num_projections)
    
    # 4. Process each file
    for i, fname in enumerate(dcm_files):
        ds = pydicom.dcmread(os.path.join(folder_path, fname))
        
        # Read and reshape pixel data
        raw_data = np.frombuffer(ds.PixelData, dtype=np.uint16).reshape((rows, cols), order='F').T
        
        # Apply HU calibration: P = (raw * m + b) / H
        calibrated_data = (raw_data.astype(np.float32) * m + b) / H
        data_array[i] = calibrated_data
        
        # Extract and correct angle
        raw_angle = ds.AngularPosition # Tag name may vary; assuming standard
        angles[i] = raw_angle
        
    # 5. Angle unwrapping and offset
    angles = np.unwrap(angles)
    angles = -angles - np.pi
    
    # 6. Compute detector bounds
    # minp = -(center_element - 0.5) * pixel_size
    # maxp = minp + shape * pixel_size
    # Note: center_element might be a vector or scalar. Assuming scalar for central element offset.
    # If it's a vector, apply per axis.
    if isinstance(center_element, (list, tuple)):
        ce_u, ce_v = center_element
    else:
        ce_u = ce_v = center_element
        
    minp_u = -(ce_u - 0.5) * pixel_size_u
    maxp_u = minp_u + cols * pixel_size_u
    
    minp_v = -(ce_v - 0.5) * pixel_size_v
    maxp_v = minp_v + rows * pixel_size_v
    
    detector_bounds = np.array([[minp_u, maxp_u], [minp_v, maxp_v]])
    
    # 7. Flying focal spot correction (First-order mean offset)
    # [G] Specific implementation depends on how source shifts are stored.
    # Assuming mean offsets are applied to angles and detector bounds.
    # This step is simplified as per notes: "Only mean offsets are applied".
    # We assume these offsets are extracted and applied here.
    # For now, we leave bounds and angles as computed, noting the correction step.
    
    # 8. Create Geometry
    # ODL ConeBeamGeometry expects:
    # - src_radius
    # - det_radius
    # - detector_shape (cols, rows)
    # - detector_spacing (pixel_size_u, pixel_size_v)
    # - angles
    # - detector_bounds (optional, or derived from spacing/shape)
    
    # Note: ODL's ConeBeamGeometry usually takes detector_shape and spacing.
    # The bounds calculation is used to define the detector plane extent.
    geometry = ConeBeamGeometry(
        src_radius=src_radius,
        det_radius=det_radius,
        detector_shape=(cols, rows),
        detector_spacing=(pixel_size_u, pixel_size_v),
        angles=angles
    )
    
    return data_array, angles, geometry
```

### Parameters
-   `indices`: Optional. Slice or list of integers to select specific projections. Default is `None` (all) [S1, unofficial].
-   `interp`: String. Interpolation method for the ray transform. Default `'linear'` [S1, unofficial].

### Edge Cases and Pitfalls
-   **Angle Definition**: DICOM angles often use a different reference frame or direction than expected by reconstruction libraries. The unwrapping and `-pi` offset are critical [S1, unofficial].
-   **Flying Focal Spot**: Full correction is complex. This method applies only first-order mean offsets, which may introduce artifacts if the focal spot shift is significant or non-linear [S1, unofficial].
-   **Pixel Center vs. Corner**: Detector bounds must be calculated from pixel centers to corners to avoid misalignment in the ray transform. Using pixel centers directly as bounds leads to incorrect ray origins [S1, unofficial].
-   **Missing Metadata**: If `HUCalibrationFactor` is missing or 1, calibration is skipped. If `ConstantRadialDistance` is not in metadata, it must be provided externally [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: Linear with respect to the number of DICOM files $N$. Reading and calibrating each file is $O(C \cdot R)$, where $C$ and $R$ are detector dimensions. Total time is $O(N \cdot C \cdot R)$.
-   **Memory Complexity**: $O(N \cdot C \cdot R)$ for the `data_array` and $O(N)$ for the `angles` array. The geometry object is $O(1)$ relative to $N$.
-   **Performance**: Depends heavily on I/O speed for reading DICOM files. NumPy operations are vectorized and efficient.
-   **Strengths**: Handles non-standard DICOM metadata, applies HU calibration, and correctly maps detector coordinates for cone-beam geometry.
-   **Limitations**: Simplified flying focal spot correction. Requires knowledge of `ConstantRadialDistance` if not in metadata. Assumes consistent detector geometry across all projections.

## Background
This method is part of the DGR (Differentiable Gaussian Rasterizer) project for CT reconstruction, specifically designed to load projection data from the Mayo Clinic dataset [S1, unofficial]. It addresses the specific geometry and metadata quirks of the CT scanners used in that dataset, enabling accurate forward projection and reconstruction using libraries like ODL and TIGRE [S1, unofficial].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
