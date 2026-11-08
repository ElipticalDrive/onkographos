# DICOM CT Projection Loading and Geometry Extraction

Also called: CT Projection Geometry Extraction & Coordinate Conversion

## Overview
This method loads raw CT projection data from a directory of DICOM files, extracts scanner geometry parameters, applies Hounsfield Unit (HU) calibration, and constructs a geometry object suitable for tomographic reconstruction [S1, unofficial]. It specifically addresses the challenge of interpreting non-standard DICOM metadata (such as angle definitions and flying focal spot offsets) to correctly map 2D detector coordinates to 3D ray paths [S1, unofficial]. It differs from standard CT loading by explicitly handling cylindrical-to-rectangular coordinate conversion for the ray transform and correcting for pixel-center vs. pixel-corner detector geometry [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `folder_path`: A directory path containing `.dcm` files [S1, unofficial].
- `indices`: Optional advanced indexing (slice or list) to load specific projection angles [S1, unofficial].
- `interp`: Interpolation method string (default `'linear'`) used for the ray transform [S1, unofficial].
- `constant_radial_distance`: Required scalar value used to compute the detector radius [S1, unofficial].

**Outputs:**
- `data_array`: A 3D NumPy array of shape `[num_projections, detector_columns, detector_rows]` with `dtype=float32`, containing calibrated projection values [S1, unofficial].
- `angles`: A 1D NumPy array of shape `[num_projections]` containing the corrected source angles in radians [S1, unofficial].
- `geometry`: A geometry object containing the source, detector, and ray configuration [S1, unofficial].

### Data Structures
1. **DICOM Dataset Object**: Standard pydicom Dataset containing tags such as `PixelData`, `RescaleSlope`, `RescaleIntercept`, `HUCalibrationFactor`, `DetectorFocalCenterRadialDistance`, `DetectorCentralElement`, `DetectorSpacing`, and angular position tags [S1, unofficial].
2. **Projection Array**: `float32` array, shape `[N, C, R]` where $N$ is the number of projections, $C$ is detector columns, $R$ is detector rows [S1, unofficial].
3. **Geometry Object**: Contains `src_radius`, `det_radius`, `detector_shape`, `detector_spacing`, `angles`, and `detector_bounds` [S1, unofficial].

### Steps
1. List and sort `.dcm` files in the input directory [S1, unofficial]. If `indices` are provided, filter the file list accordingly [S1, unofficial].
2. For each file, read DICOM tags for detector rows/cols, HU calibration factor, angular positions, detector spacing, radial distances, and source shifts [S1, unofficial].
3. Extract `PixelData` as `uint16`, reshape to `[rows, cols]` in Fortran order, and transpose to `[cols, rows]` [S1, unofficial].
4. Apply HU calibration: $P = \frac{(P_{raw} \cdot m + b)}{H}$, where $m$ is `RescaleSlope`, $b$ is `RescaleIntercept`, and $H$ is `HUCalibrationFactor` [S1, unofficial]. Store the result in `data_array` [S1, unofficial].
5. Extract angles, unwrap them, and shift by $-\pi$: `angles = -np.unwrap(angles) - np.pi` [S1, unofficial].
6. Compute pixel sizes and detector bounds, correcting for pixel center vs. corner: `minp = -(np.array(DetectorCentralElement) - 0.5) * pixel_size`, `maxp = minp + shape * pixel_size` [S1, unofficial].
7. Extract `src_radius` from `DetectorFocalCenterRadialDistance` and compute `det_radius = ConstantRadialDistance - src_radius` [S1, unofficial].
8. Compute pitch if missing from metadata: $pitch = \frac{\Delta axial\_pos}{(\theta_{max} - \theta_{min}) / (2\pi)}$ [S1, unofficial].
9. Apply flying focal spot corrections to angles and detector bounds using mean offsets as first-order corrections [S1, unofficial].
10. Convert ray transform range coordinates from cylindrical to rectangular: $u = d \cdot \arctan(\text{up} / d)$, $v = \frac{d}{\sqrt{d^2 + \text{up}^2}} \cdot \text{vp}$, where $d = \text{src\_radius} + \text{det\_radius}$ [S1, unofficial].
11. Assemble the geometry object with the computed parameters [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import pydicom
import os

def load_dicom_projections_and_geometry(folder_path, indices=None, interp='linear', constant_radial_distance=None):
    # 1. List and sort .dcm files
    dcm_files = sorted([f for f in os.listdir(folder_path) if f.endswith('.dcm')])
    if indices is not None:
        dcm_files = [dcm_files[i] for i in indices]
    num_projections = len(dcm_files)
    if num_projections == 0:
        raise ValueError("No DICOM files found.")

    # 2. Read first file for static geometry and calibration
    ds0 = pydicom.dcmread(os.path.join(folder_path, dcm_files[0]))
    rows, cols = ds0.Rows, ds0.Columns
    m = getattr(ds0, 'RescaleSlope', 1.0)
    b = getattr(ds0, 'RescaleIntercept', 0.0)
    H = getattr(ds0, 'HUCalibrationFactor', 1.0)
    src_radius = ds0.DetectorFocalCenterRadialDistance
    det_spacing = ds0.DetectorSpacing
    center_element = ds0.DetectorCentralElement
    pixel_size = det_spacing[0] if isinstance(det_spacing, (list, tuple)) else det_spacing

    # 3. Initialize output arrays
    data_array = np.zeros((num_projections, cols, rows), dtype=np.float32)
    angles = np.zeros(num_projections)

    # 4. Process each file
    for i, fname in enumerate(dcm_files):
        ds = pydicom.dcmread(os.path.join(folder_path, fname))
        # Extract PixelData as uint16, reshape to [rows, cols] in Fortran order, transpose
        raw_data = np.frombuffer(ds.PixelData, dtype=np.uint16).reshape((rows, cols), order='F').T
        # Apply rescale
        proj_array = (raw_data.astype(np.float32) * m + b) / H
        data_array[i] = proj_array
        # Extract raw angles
        angles[i] = ds.AngularPosition

    # 5. Extract angles: unwrap and shift by -pi
    angles = -np.unwrap(angles) - np.pi

    # 6. Compute pixel sizes and min/max corners correcting for pixel center vs corner
    center_arr = np.array(center_element)
    minp = -(center_arr - 0.5) * pixel_size
    maxp = minp + np.array([cols, rows]) * pixel_size
    detector_bounds = np.stack([minp, maxp], axis=0)

    # 7. Compute source and detector radii
    det_radius = constant_radial_distance - src_radius
    d = src_radius + det_radius

    # 8. Compute pitch if missing
    # pitch = (axial_pos_diff) / ((max_angles - min_angles) / (2 * np.pi))
    # Sources do not specify exact axial position extraction; compute if metadata is available.

    # 9. Apply flying focal spot corrections
    # Sources do not specify exact source shift extraction; apply mean offsets to angles and detector bounds here.

    # 10. Convert ray transform range coordinates from cylindrical to rectangular
    # up, vp are cylindrical detector coordinates
    # u = d * np.arctan(up / d)
    # v = d / np.sqrt(d**2 + up**2) * vp

    # 11. Assemble geometry object
    # geometry = odl.tomo.ConeBeamGeometry(...) or custom CTGeometry(...)
    return data_array, angles, detector_bounds, d
```

### Parameters
- `indices`: Optional advanced indexing (slice or list) to load specific projection angles [S1, unofficial].
- `interp`: Interpolation method string for the ray transform [S1, unofficial].

### Edge Cases and Pitfalls
- **Angle Definition**: DICOM angles use a different definition requiring unwrapping and offset [S1, unofficial].
- **Flying Focal Spot**: Full correction is complex and only partially handled with first-order corrections to detector position and radial offset [S1, unofficial].
- **Pixel Center vs. Corner**: Detector bounds must be calculated from pixel centers to corners to avoid misalignment in the ray transform [S1, unofficial].
- **Missing Metadata**: If `HUCalibrationFactor` is missing or 1, calibration is skipped [S1, unofficial]. If `ConstantRadialDistance` is not in metadata, it must be provided externally [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of DICOM files $N$ [S1, unofficial]. Reading and calibrating each file is $O(C \cdot R)$, where $C$ and $R$ are detector dimensions [S1, unofficial]. Total time is $O(N \cdot C \cdot R)$ [S1, unofficial].
- **Memory Complexity**: $O(N \cdot C \cdot R)$ for the `data_array` and $O(N)$ for the `angles` array [S1, unofficial]. The geometry object is $O(1)$ relative to $N$ [S1, unofficial].
- **Performance**: Depends heavily on I/O speed for reading DICOM files [S1, unofficial]. NumPy operations are vectorized and efficient [S1, unofficial].
- **Strengths**: Handles non-standard DICOM metadata, applies HU calibration, and correctly maps detector coordinates for cone-beam geometry [S1, unofficial].
- **Limitations**: Simplified flying focal spot correction [S1, unofficial]. Requires knowledge of `ConstantRadialDistance` if not in metadata [S1, unofficial]. Assumes consistent detector geometry across all projections [S1, unofficial].

## Background
This method is part of the DGR (Differentiable Gaussian Rasterizer) project for CT reconstruction [S1, unofficial]. It is specifically designed to load projection data from the Mayo Clinic dataset [S1, unofficial]. It addresses the specific geometry and metadata quirks of the CT scanners used in that dataset [S1, unofficial]. It enables accurate forward projection and reconstruction using libraries like ODL and TIGRE [S1, unofficial].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
