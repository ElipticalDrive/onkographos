# VTK Image Data Construction from NumPy

Also called: NIfTI Loading, NumPy to VTK Image Data Conversion, Volume Data Import and Spacing Configuration, vtkImageImportFromArray

## Overview
This method imports volume data from NumPy arrays or NIfTI files into VTK, ensuring correct memory layout, data spacing, and origin configuration for normalized volume rendering [S2, unofficial]. It differs from generic array conversion by explicitly handling geometric metadata to prevent visual artifacts such as volumes appearing too thin [S2, unofficial].

## How it works
**Inputs:**
1. Source data: A NumPy array, NIfTI file, or HDR file [S1, S2, unofficial].
2. Metadata: Spacing tuple $(s_x, s_y, s_z)$ and origin tuple $(o_x, o_y, o_z)$ [S1, S2, unofficial].

**Outputs:**
1. A `vtkImageData` object containing the volume data as scalars [S1, unofficial].

**Data Structures:**
* NumPy Array: The source volume data [S1, S2, unofficial].
* VTK Array: A `vtkUnsignedCharArray` or `vtkUnsignedShortArray` holding the flattened data [S1, S2, unofficial].
* vtkImageData: The VTK object representing the structured volume [S1, S2, unofficial].

**Steps:**
1. Load and preprocess data based on format [S2, unofficial].
2. Flatten the NumPy array using Fortran order ('F') [S1, S2, unofficial].
3. Create a `vtkImageImportFromArray` object [S2, unofficial].
4. Set the flattened array on the import object [S2, unofficial].
5. Configure spacing and origin on the import object [S2, unofficial].
6. Enable integer-to-unsigned-short conversion if needed [S2, unofficial].
7. Create the final `vtkImageData` object, set dimensions, and attach scalars [S1, S2, unofficial].

**Formulas:**
* NIfTI spacing: $s_x = \text{header}['\text{pixdim}'][1]$, $s_y = \text{header}['\text{pixdim}'][2]$, $s_z = \text{header}['\text{pixdim}'][3]$ [S2, unofficial].
* NIfTI origin: $o_x = \text{header}['\text{dim}'][0] \times \text{header}['\text{pixdim}'][0]$, $o_y = \text{header}['\text{dim}'][1] \times \text{header}['\text{pixdim}'][1]$, $o_z = \text{header}['\text{dim}'][2] \times \text{header}['\text{pixdim}'][2]$ [S2, unofficial].
* 4D volume summation: $A = \sum_{i=0}^{N-1} D[i]$ [S2, unofficial].
* 5D tensor summation: $A = \sum_{i=0}^{N-1} D[i][0]$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import vtk

def numpy_to_vtk_image_data(array, spacing, origin):
    # Step 1: Flatten in Fortran order for correct memory layout [S1, S2, unofficial]
    flat_array = array.flatten(order='F')
    
    # Step 2: Create VTK import object [S2, unofficial]
    img_import = vtk.vtkImageImportFromArray()
    
    # Step 3: Set the numpy array [S2, unofficial]
    img_import.SetArray(flat_array)
    
    # Step 4: Set data spacing from header or provided tuple [S2, unofficial]
    img_import.GetImport().SetDataSpacing(spacing)
    
    # Step 5: Set data origin from header dimensions and pixel dimensions [S2, unofficial]
    img_import.GetImport().SetDataOrigin(origin)
    
    # Step 6: Convert integer data to unsigned short if applicable [S2, unofficial]
    img_import.ConvertIntToUnsignedShortOn()
    
    # Step 7: Update import pipeline to generate output [G]
    img_import.Update()
    
    # Step 8: Create vtkImageData object and set dimensions [S1, S2, unofficial]
    img_data = vtk.vtkImageData()
    img_data.SetDimensions(array.shape[0], array.shape[1], array.shape[2])
    img_data.SetSpacing(spacing)
    img_data.SetOrigin(origin)
    
    # Step 9: Attach the converted array as scalars to point data [S1, S2, unofficial]
    img_data.GetPointData().SetScalars(img_import.GetOutput())
    
    return img_data
```

**Edge Cases and Pitfalls:**
* Incorrect spacing/origin causes volume to appear too thin [S2, unofficial].
* C-order flattening causes incorrect voxel mapping; Fortran order is required [S1, S2, unofficial].
* Integer to unsigned short conversion requires care [S2, unofficial].

## Subtopics

### [Data Contiguity Check](18253-data-contiguity-check.md)
This method validates that a NumPy array meets the strict memory layout and type requirements of downstream volume rendering or tomography engines, specifically ensuring C-contiguity and `float32` precision. It differs from general array validation by strictly enforcing a contiguous memory block without copying unless explicitly permitted, preventing silent data corruption in high-performance C/C++ backends. It serves as a prerequisite for [Memory-In Task Creation](20984-memory-in-task-creation.md) to ensure in-memory data structures are compatible with the processing engine.

### VTK Volume Rendering from Memory
This technique renders 3D data stored in a one-dimensional array as a uniform 3D grid using VTK without copying the data, by allocating VTK scalars and casting the pointer to the underlying memory [S3, unofficial]. It sets the dimensions of the image grid using `SetDimensions(dx, dy, dz)`, allocates scalars for the image using `AllocateScalars` with the data type and number of components per cell, and gets a pointer to the scalar data using `GetScalarPointer()` cast to the appropriate type to populate the grid directly [S3, unofficial]. The grid size is calculated as $dx \times dy \times dz \times 3$ for vector fields [S3, unofficial].

## Operating characteristics
* Time Complexity: $O(N)$ where $N$ is the number of voxels, for flattening and conversion [S1, S2, unofficial].
* Memory Complexity: $O(N)$ for the flattened array and VTK array [S1, S2, unofficial].
* Strengths: Correctly handles geometric metadata, preventing rendering artifacts [S2, unofficial].
* Limitations: Requires manual handling of 4D/5D data summation [S2, unofficial].
* When to Use: When importing volume data into VTK for rendering, especially when geometric accuracy is critical [S2, unofficial].
* When Not to Use: When working with data that does not require geometric metadata or when using a different rendering framework [G].

## Background
This method is derived from implementations in medical imaging viewers and volume rendering tools [S1, S2, unofficial]. It addresses the specific needs of DWI and NIfTI data, which often require summing multiple volumes or components to create a single scalar volume for rendering [S2, unofficial].

## Sources
[S1] caitlin-leonard/medviewer, https://github.com/caitlin-leonard/medviewer, fetched 2026-10-27 (unofficial)
[S2] chapering/PyVolRender, https://github.com/chapering/PyVolRender, fetched 2026-10-27 (unofficial)
[S3] VTK 3D Volume Rendering from Memory, https://stackoverflow.com/questions/16534101/vtk-3d-volume-rendering-from-memory, fetched 2026-10-28 (unofficial)
