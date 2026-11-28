# Volume of Interest (VOI) Extraction with Sampling

Also called: Subvolume Extraction, vtkExtractVOI

## Overview
Builds on [Volume Subset Extraction](5602-volume-subset-extraction.md), this technique extracts a rectangular sub-region from a 3D volume dataset and optionally downsamples it using integer sampling rates [S1, unofficial]. Unlike continuous interpolation-based cropping, it uses integer strides for sampling, which is computationally simpler but limits resolution adjustments to integer factors [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset (e.g., `vtkImageData` in VTK) defined by dimensions, extent, and scalar type [S1, unofficial].
*   **Output:** A new 3D volume dataset representing the extracted sub-region, with dimensions determined by the VOI bounds and sampling rates [S1, unofficial].

**Data Structures**
The input volume is a 3D dataset with a defined extent [S1, unofficial]. The output volume retains the input's scalar type and component count but has reduced dimensions [S1, unofficial]. Internal memory layout details are Not given in the sources [S1, unofficial].

**Parameters**
1.  **VOI Bounds:** Six integers $(minX, maxX, minY, maxY, minZ, maxZ)$ defining the inclusive range of the sub-volume within the original dataset dimensions [S1, unofficial].
2.  **Sampling Rate:** Three integers $(sx, sy, sz)$ defining the stride for sampling along each axis [S1, unofficial].

**Steps**
1.  Create an instance of the extraction filter (e.g., `vtkExtractVOI`) [S1, unofficial].
2.  Connect the input volume data to the filter using the `SetInput` method [S1, unofficial].
3.  Define the Volume of Interest bounds by calling `SetVOI` with six integer arguments $(minX, maxX, minY, maxY, minZ, maxZ)$ representing the inclusive range of the sub-volume [S1, unofficial].
4.  Define the sampling rate by calling `SetSampleRate` with three integer arguments $(sx, sy, sz)$ defining the stride along each axis, where $1$ means no downsampling and values $>1$ skip voxels [S1, unofficial].
5.  Execute the filter (e.g., via `Update()` in VTK) to generate the output data [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_voi(input_volume, bounds, sample_rate):
    # bounds: (minX, maxX, minY, maxY, minZ, maxZ)
    # sample_rate: (sx, sy, sz)
    # input_volume: 3D list or array-like structure
    minX, maxX, minY, maxY, minZ, maxZ = bounds
    sx, sy, sz = sample_rate
    
    out_x = (maxX - minX) // sx + 1
    out_y = (maxY - minY) // sy + 1
    out_z = (maxZ - minZ) // sz + 1
    
    output_volume = [[[None for _ in range(out_x)] for _ in range(out_y)] for _ in range(out_z)]
    
    for iz in range(out_z):
        for iy in range(out_y):
            for ix in range(out_x):
                src_x = minX + ix * sx
                src_y = minY + iy * sy
                src_z = minZ + iz * sz
                output_volume[iz][iy][ix] = input_volume[src_z][src_y][src_x]
    return output_volume
```

**Edge Cases and Pitfalls**
*   **Integer Constraints:** Sampling rates must be integers; fractional sampling rates are not supported [S1, unofficial]. This leads to discrete steps in resolution rather than continuous scaling [S1, unofficial].
*   **Bounds Validation:** The specified VOI bounds must be within the valid extent of the input volume; out-of-bounds indices may cause errors or undefined behavior [G].
*   **Degenerate VOI:** If $minX > maxX$ or similar for other axes, the resulting volume may be empty or invalid [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Computationally simple and fast due to the lack of interpolation [S1, unofficial].
*   **Limitations:** Resolution adjustments are limited to integer factors, preventing fine-grained downsampling [S1, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
This method is implemented via the VTK filter `vtkExtractVOI` [S1, unofficial]. It is part of the Visualization Toolkit (VTK) library for 3D computer graphics, image processing, and visualization [S1, unofficial].

## Sources
[S1] Isosurface Generation using VTK: Trying Different Volumes of Interest, https://www.evl.uic.edu/aspale/cs526/final/3-5-4-0.htm, fetched 2026-10-30 (unofficial)
