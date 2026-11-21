# DICOM Volume Loading and Coordinate Reconstruction

Also called: DICOM File Loading and Stacking, DICOM Loader, DICOM Volume Loading and Normalization, DICOM Volume Reconstruction Loading & Density Conversion, DICOM to Density Conversion, Volume DICOM Loading and Rescaling

## Overview
This method loads a 3D volume from a series of DICOM slice files, converts raw pixel values to physical density (Hounsfield Units normalized to water = 1.0), and reconstructs the correct voxel partition by accounting for DICOM coordinate conventions and spacing [S2, S3, unofficial]. It sets itself apart from generic loaders by explicitly handling the conversion from DICOM's middle-of-pixel coordinate system to the corner-point convention required by many rendering engines (e.g., ODL), and by normalizing density relative to water rather than just scaling to [0, 1] [S2, unofficial].

## How it works
**Inputs:** A directory of DICOM files (typically `.dcm` or `.IMA`), sorted by filename.
**Outputs:** A 3D volume array (density values), voxel size (scale), and world-space bounds (min/max points).

**Data Structures:**
*   `volumeData`: 3D array of shape `(num_slices, height, width)` or `(width, height, num_slices)` depending on final layout. Dtype `float32` for density, or `uint16` for raw storage [S2, S4, unofficial].
*   `scale`: Array `[pixelSpacing[0], pixelSpacing[1], sliceThickness]` in millimeters [S2, S3, unofficial].
*   `bounds`: `min_pt` and `max_pt` defining the world-space bounding box [S2, unofficial].

**Steps:**

1.  **File Selection and Sorting:**
    Load all DICOM files from the specified directory. Filter for extensions `.dcm` or `.IMA`. Sort the file list alphabetically to ensure correct slice ordering [S2, S4, unofficial].

2.  **Metadata Extraction:**
    Read the first DICOM file to determine base dimensions and pixel representation [S4, unofficial].
    *   `rows`: Number of rows (height).
    *   `cols`: Number of columns (width).
    *   `bitsAllocated`: Bits per sample.
    *   `pixelRepresentation`: 0 for unsigned, 1 for signed [S3, unofficial].
    *   `pixelSpacing`: `[spacing_y, spacing_x]` (Note: DICOM order is often Y then X) [S2, unofficial].
    *   `sliceThickness`: Distance between slices [S2, unofficial].
    *   `RescaleSlope` and `RescaleIntercept`: Used for HU conversion [S2, unofficial].
    *   `DataCollectionCenterPatient`: 3D point `(x, y, z)` for the center of the slice [S2, unofficial].
    *   `ReconstructionTargetCenterPatient`: Optional target center [S2, unofficial].

3.  **Volume Initialization:**
    Calculate `num_slices` from the number of files.
    Initialize a buffer. If storing raw data, use `Uint16Array` of size `rows * cols * num_slices` [S3, unofficial]. If storing density, use `float32` array of shape `(num_slices, rows, cols)` [S4, unofficial].

4.  **Pixel Data Loading and Orientation Correction:**
    Iterate through each sorted DICOM file:
    *   Read the pixel array.
    *   Determine `bytesPerVoxel = bitsAllocated / 8`. If `bytesPerVoxel != 2`, throw an error (unsupported format) [S3, unofficial].
    *   Cast pixel data to `float32`.
    *   Reshape to `[cols, rows]` in C-order [S2, unofficial].
    *   **Orientation Fix:** DICOM images are often stored upside-down or rotated relative to standard image coordinates. Apply a -90 degree rotation: `data_array = np.rot90(data_array, k=-1)` [S2, S4, unofficial].
    *   Store in the volume buffer.

5.  **HU to Density Conversion:**
    Convert raw pixel values to Hounsfield Units (HU), then to physical density where water (0 HU) = 1.0 [S2, unofficial].
    Formula:
    $$ \text{HU} = \text{RescaleSlope} \cdot \text{raw\_value} + \text{RescaleIntercept} $$
    $$ \text{density} = \frac{\text{HU} + 1000}{1000} $$
    Combined:
    $$ \text{density} = \frac{\text{RescaleSlope} \cdot \text{raw\_value} + \text{RescaleIntercept} + 1000}{1000} $$
    [S2, unofficial].

    *Note: Some implementations normalize to [0, 255] for visualization, but for physical volume rendering, the density formula above is required [S2, unofficial].*

6.  **Coordinate Reconstruction and Bounds Calculation:**
    Compute the world-space bounding box. DICOM coordinates represent the *center* of the voxel, while many renderers expect *corner* coordinates [S2, unofficial].

    *   **Voxel Size:**
        $$ \text{voxel\_size} = [\text{pixelSpacing}[0], \text{pixelSpacing}[1], \text{sliceThickness}] $$
        [S2, unofficial].

    *   **Slice Distance:**
        If `num_slices > 1`:
        Calculate the distance between the first two slices using the Z-component of `DataCollectionCenterPatient`:
        $$ \text{slice\_distance} = | \text{DataCollectionCenterPatient}_1.z - \text{DataCollectionCenterPatient}_0.z | $$
        [S2, unofficial].
        If `num_slices == 1`:
        Approximate `slice_distance = sliceThickness` [S2, unofficial].

    *   **Z-Axis Bounds:**
        Use the first and last slice centers. Adjust for the corner-point convention by adding/subtracting half the slice distance [S2, unofficial]:
        $$ \text{min\_pt}[2] = \text{DataCollectionCenterPatient}_{\text{first}}.z - \frac{\text{slice\_distance}}{2} $$
        $$ \text{max\_pt}[2] = \text{DataCollectionCenterPatient}_{\text{last}}.z + \frac{\text{slice\_distance}}{2} $$
        [S2, unofficial].
        *Pitfall:* If the Z-computation fails or bounds are inverted, reverse the min/max assignment (use last slice for min, first for max) and log an error [S2, unofficial].

    *   **X/Y Axis Bounds:**
        Calculate based on `ReconstructionTargetCenterPatient` or derived from image dimensions and pixel spacing.
        *Pitfall:* DICOM Y-axis (Axis 1) convention is often reversed compared to image coordinates. Swap min and max values for the Y-axis if necessary [S2, unofficial].

    *   **Final Bounds:**
        $$ \text{min\_pt} = [x_{min}, y_{min}, z_{min}] $$
        $$ \text{max\_pt} = [x_{max}, y_{max}, z_{max}] $$
        [S2, unofficial].

7.  **Transposition (Optional/Implementation Dependent):**
    Some pipelines transpose the volume from `(rows, cols, slices)` to `(slices, rows, cols)` or `(1, 2, 0)` layout to match renderer expectations [S2, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import pydicom
import os
import glob

def load_dicom_volume(directory, extension='.dcm'):
    # 1. File Selection
    files = sorted(glob.glob(os.path.join(directory, f'*{extension}')))
    if not files:
        raise ValueError("No DICOM files found")

    # 2. Metadata Extraction from first file
    ds_first = pydicom.dcmread(files[0])
    rows = int(ds_first.Rows)
    cols = int(ds_first.Columns)
    pixel_spacing = list(ds_first.PixelSpacing) # [y, x]
    slice_thickness = float(ds_first.SliceThickness)
    rescale_slope = float(ds_first.RescaleSlope)
    rescale_intercept = float(ds_first.RescaleIntercept)
    bits_allocated = int(ds_first.BitsAllocated)
    
    if bits_allocated != 16:
        raise ValueError("Unsupported bitsAllocated. Expected 16.")

    num_slices = len(files)
    
    # 3. Initialize Volume
    # Shape: (num_slices, rows, cols)
    volume = np.zeros((num_slices, rows, cols), dtype=np.float32)

    # 4. Load and Process Slices
    for i, f in enumerate(files):
        ds = pydicom.dcmread(f)
        # Read pixel data
        pixel_array = ds.pixel_array
        
        # Cast to float
        data = pixel_array.astype(np.float32)
        
        # Orientation Correction: Reshape to (cols, rows) and rotate -90 deg
        # Note: pixel_array shape is (rows, cols)
        data = data.reshape((cols, rows))
        data = np.rot90(data, k=-1) # Rotate -90 degrees
        
        # 5. HU to Density Conversion
        hu = rescale_slope * data + rescale_intercept
        density = (hu + 1000.0) / 1000.0
        
        volume[i, :, :] = density

    # 6. Coordinate Reconstruction
    # Voxel Size
    voxel_size = [pixel_spacing[0], pixel_spacing[1], slice_thickness]
    
    # Slice Distance
    if num_slices > 1:
        ds_second = pydicom.dcmread(files[1])
        z1 = float(ds_first.DataCollectionCenterPatient[2])
        z2 = float(ds_second.DataCollectionCenterPatient[2])
        slice_distance = abs(z2 - z1)
    else:
        slice_distance = slice_thickness

    # Z Bounds (Corner Point Convention)
    z_first = float(ds_first.DataCollectionCenterPatient[2])
    ds_last = pydicom.dcmread(files[-1])
    z_last = float(ds_last.DataCollectionCenterPatient[2])
    
    min_pt_z = z_first - (slice_distance / 2.0)
    max_pt_z = z_last + (slice_distance / 2.0)
    
    # X/Y Bounds (Simplified: assuming origin at 0,0 for center, adjust based on specific DICOM tags if available)
    # Using ReconstructionTargetCenterPatient if available, else estimate from spacing
    # Note: Y-axis convention may need swapping [S2, unofficial]
    width_mm = cols * pixel_spacing[1]
    height_mm = rows * pixel_spacing[0]
    
    # Assuming center of image is at (0,0) in patient space for simplicity if tags missing
    # Real implementation should use ImagePositionPatient or ReconstructionTargetCenterPatient
    min_pt_x = -width_mm / 2.0
    max_pt_x = width_mm / 2.0
    min_pt_y = -height_mm / 2.0
    max_pt_y = height_mm / 2.0
    
    # Swap Y min/max if DICOM convention requires it [S2, unofficial]
    # This depends on specific viewer/renderer requirements. 
    # Generic swap:
    min_pt_y, max_pt_y = max_pt_y, min_pt_y

    min_pt = [min_pt_x, min_pt_y, min_pt_z]
    max_pt = [max_pt_x, max_pt_y, max_pt_z]

    return volume, voxel_size, min_pt, max_pt
```

## Subtopics

### DICOM Volume Texture Preparation
Prepares the loaded volume for GPU rendering by quantizing voxel values into a specific window range (lower/upper thresholds in HU) and integer-downsampling if any axis exceeds 256 voxels. Physical size in millimeters is stored separately from texture dimensions [S1, unofficial].

### DICOM to MHA Conversion
Converts DICOM series to MetaImage (MHD/RAW) format. Reads the first file for dimensions/spacing, initializes a numpy array, iterates through files to populate it, transposes axes from `(x, y, z)` to `(z, y, x)` to match DICOM storage, and saves using SimpleITK [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R \cdot C)$, where $N$ is the number of slices, $R$ is rows, $C$ is columns. Linear with respect to total voxel count.
*   **Memory Complexity:** $O(N \cdot R \cdot C)$ for storing the volume array.
*   **Performance:** Loading speed depends on I/O throughput and DICOM parsing overhead. Conversion to density is computationally cheap. Orientation correction (rotation) adds minor overhead.
*   **Strengths:** Handles standard medical imaging conventions (HU, spacing, orientation). Provides physically meaningful density values.
*   **Limitations:** Assumes consistent slice ordering and spacing. Fails if `bytesAllocated` is not 16. Requires careful handling of coordinate system differences (DICOM vs. Renderer).
*   **When to use:** When loading CT data for physical volume rendering or MPR.
*   **When not to use:** For non-CT modalities without HU scaling, or when real-time streaming is required without full volume load.

## Background
DICOM (Digital Imaging and Communications in Medicine) is the standard for medical imaging data. Loading DICOM volumes requires parsing specific tags (PixelSpacing, SliceThickness, RescaleSlope) and handling the coordinate system differences between DICOM (patient-centered, middle-of-voxel) and typical graphics APIs (corner-of-voxel). This method bridges that gap, enabling accurate volume rendering of medical data. It is commonly used in medical visualization software, research prototypes, and clinical PACS viewers.

## Sources
[S1] feat(body-exposure): render a DICOM series in 3D, not a description of one, https://github.com/rizkyazhar486/Panacea/pull/1585, fetched 2026-10-26 (unofficial)
[S2] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S3] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S4] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
[S5] comedate/VolumeRendering, https://github.com/comedate/VolumeRendering, fetched 2026-10-27 (unofficial)
