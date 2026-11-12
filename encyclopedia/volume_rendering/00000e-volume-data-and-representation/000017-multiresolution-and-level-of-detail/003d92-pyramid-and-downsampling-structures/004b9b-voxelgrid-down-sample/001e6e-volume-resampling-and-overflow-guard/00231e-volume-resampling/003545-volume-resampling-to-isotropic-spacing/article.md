# Volume Resampling to Isotropic Spacing

Also called: isotropic resampling, spacing normalization

## Overview
This method resamples a volume with arbitrary anisotropic voxel spacing into a new volume where all axes share a single, uniform spacing value. It solves the problem of inconsistent physical scales across axes, which complicates downstream operations like filtering or rendering that assume uniform grid metrics. Unlike [Area-Averaged Resampling](3142-area-averaged-resampling.md), which focuses on preserving integrated intensity during downsampling, this method strictly enforces geometric isotropy by calculating output dimensions based on physical extent rather than simple integer division of voxel counts.

## How it works
**Inputs and Outputs:**
*   **Input:** A `VolumeData` object containing spatial dimensions, per-axis spacing, origin, direction matrix, and a voxel buffer.
*   **Output:** A new `VolumeData` object with isotropic spacing and recalculated dimensions.

**Data Structures:**
The input `VolumeData` layout is defined as follows [S1, unofficial]:
*   `width`, `height`, `depth`: `size_t` (unsigned integer types representing voxel counts along X, Y, Z).
*   `spacingX`, `spacingY`, `spacingZ`: `float` (physical size of one voxel along each axis).
*   `voxels`: `std::vector<float>` (linear buffer of voxel values, typically row-major or column-major depending on the source format, but treated as a contiguous block here).
*   `SpatialGeometry`:
    *   `origin`: `float[3]` (physical coordinates of the voxel at index (0,0,0)).
    *   `direction`: `float[9]` (3x3 rotation/scaling matrix, stored in row-major order).

**Parameters:**
*   `kTargetSpacing`: `float`. The desired uniform spacing for all axes. A typical value is `1.0F` [S1, unofficial].

**Steps:**
1.  **Validation:** Check for degenerate inputs. If any input dimension is zero, or if any spacing value is non-finite (NaN/Inf) or zero, throw `std::invalid_argument` [S1, unofficial].
2.  **Physical Extent Calculation:** For each axis $i \in \{x, y, z\}$, calculate the physical extent of the volume. The extent is the distance from the start of the first voxel to the start of the last voxel:
    $$ \text{extent}_i = (\text{inputSize}_i - 1) \times \text{inputSpacing}_i $$
3.  **Output Dimension Calculation:** Compute the number of voxels required to cover this extent at the target isotropic spacing. The formula for each axis is [S1, unofficial]:
    $$ \text{outputSize}_i = \max\left(1, \text{round}\left(\frac{\text{extent}_i}{\text{kTargetSpacing}}\right) + 1\right) $$
    *   `round`: Rounds to the nearest integer.
    *   The `+ 1` accounts for the fact that $N$ voxels span $N-1$ intervals.
    *   The `max(1, ...)` ensures the output dimension is at least 1, even if the physical extent is zero or negative due to precision issues.
4.  **Overflow Check:** Verify that the calculated output dimensions do not cause integer overflow when used to allocate memory or index buffers. Use checked multiplication if necessary [S1, unofficial].
5.  **Resampling (Implementation Note):** While the notes describe the geometry calculation, the actual resampling of voxel values is typically performed by mapping the input `VolumeData` to an intermediate image format (e.g., ITK Image), performing the resampling using a library's resampler (which handles interpolation), and mapping back. The geometry (origin, direction, spacing) of the output volume is set to the new isotropic spacing, the calculated output size, and the preserved origin/direction [S1, unofficial].
6.  **Output Construction:** Create the new `VolumeData` object with the calculated `outputSize` values, `kTargetSpacing` for all axes, the original `origin` and `direction`, and the resampled voxel buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resample_to_isotropic(volume: VolumeData, target_spacing: float = 1.0) -> VolumeData:
    # 1. Validation
    if volume.width == 0 or volume.height == 0 or volume.depth == 0:
        raise ValueError("Input dimensions must be non-zero")
    
    spacings = [volume.spacingX, volume.spacingY, volume.spacingZ]
    for s in spacings:
        if not is_finite(s) or s == 0:
            raise ValueError("Spacing must be finite and non-zero")

    # 2. Calculate output dimensions
    output_sizes = []
    for i in range(3):
        input_size = [volume.width, volume.height, volume.depth][i]
        input_spacing = spacings[i]
        
        # Physical extent: (N-1) * spacing
        extent = (input_size - 1) * input_spacing
        
        # Output size: round(extent / target) + 1, min 1
        raw_size = round(extent / target_spacing) + 1
        output_size = max(1, raw_size)
        output_sizes.append(output_size)
    
    # 3. Overflow check (conceptual)
    # In C++, this would use checkedMultiply or similar to ensure 
    # output_sizes[0] * output_sizes[1] * output_sizes[2] fits in size_t
    # and does not exceed memory limits. [G]

    # 4. Resampling
    # The actual voxel value interpolation is handled by the underlying 
    # ITK resampler or equivalent library call. The geometry is updated 
    # as follows:
    new_volume = VolumeData()
    new_volume.width = output_sizes[0]
    new_volume.height = output_sizes[1]
    new_volume.depth = output_sizes[2]
    new_volume.spacingX = target_spacing
    new_volume.spacingY = target_spacing
    new_volume.spacingZ = target_spacing
    new_volume.origin = volume.origin
    new_volume.direction = volume.direction
    
    # Perform resampling from volume.voxels into new_volume.voxels
    # using the input geometry and output geometry.
    # This step is implementation-dependent but typically involves:
    # 1. Converting input VolumeData to ITK Image<float, 3>
    # 2. Using itkResampleImageFilter with the new geometry
    # 3. Converting the result back to VolumeData
    new_volume.voxels = resample_voxels(volume.voxels, volume, new_volume)
    
    return new_volume
```

## Subtopics

### Area-Averaged Resampling
Focuses on preserving the integral of the volume data during downsampling, often used for creating mipmaps. Choose this when maintaining energy/intensity conservation is critical, rather than strict geometric isotropy. [Link](3142-area-averaged-resampling.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of input and output voxels, $O(N_{in} + N_{out})$, assuming the underlying resampling library (e.g., ITK) uses linear or nearest-neighbor interpolation. Higher-order interpolators increase the constant factor.
*   **Memory Complexity:** $O(N_{out})$ for the output buffer. The input buffer is retained during the process.
*   **Strengths:** Guarantees uniform voxel spacing, which simplifies subsequent processing steps like Fourier transforms or uniform grid-based rendering.
*   **Limitations:** The output dimensions are determined by physical extent, which may result in a significantly larger or smaller voxel count than the input, potentially leading to high memory usage or loss of detail if the target spacing is very small.
*   **When to use:** When downstream algorithms require isotropic voxels (e.g., certain medical imaging analyses, isotropic rendering).
*   **When not to use:** When preserving the exact original voxel count or aspect ratio is more important than geometric uniformity.

## Background
This method is implemented in the `qt-viewer-pro` project, specifically within the `ItkVolumeConverter` and volume loading modules. It leverages the Insight Toolkit (ITK) for robust image processing and geometry handling. The approach reflects standard practices in medical image processing where anisotropic scans (common in MRI/CT) are resampled to isotropic grids for visualization and analysis.

## Sources
[S1] wpop/qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
