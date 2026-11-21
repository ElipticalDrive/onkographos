# Volume Resampling

Also called: Data Representation and Processing, Data Resampling, Isotropic Volume Resampling, Volume Resampling Comparison, Volumetric Data Resampling, data resampling tool

## Overview
Volume resampling is a preprocessing technique that converts unstructured particle data (such as SPH) or arbitrary-resolution volumetric data into a structured 3D grid to enable fast direct volume rendering [S1][S4, unofficial]. It solves the problem of expensive per-particle evaluation during rendering by shifting the computational load to an offline or pre-rendering phase [S1]. It distinguishes itself from direct particle rendering by trading long preprocessing times for significantly faster rendering speeds, though it struggles with precise reconstruction of phase interfaces due to discretization artifacts [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Source volumetric data (e.g., SPH particles, CT scans) with original dimensions, spacing, and origin [S1][S2, unofficial][S3].
*   **Output:** A resampled 3D grid (volume) with target dimensions, isotropic spacing, and preserved geometry [S1][S2, unofficial][S3].

**Data Structures:**
The output is typically stored as a 3D array or texture [S3]. For GPU rendering, power-of-two-size textures are often used to maximize performance [S3]. In ITK-based implementations, the data is represented as an `itk::Image` with specific spatial geometry metadata [S2, unofficial].

**Steps:**
1.  **Data Conversion:** Convert the input volume data into a processing-friendly format, such as an ITK image [S2, unofficial].
2.  **Dimension Calculation:** Calculate the output dimensions for each axis by calling `resampledDimension` with the input size, input spacing, and `kTargetSpacing`, casting results to `std::size_t` [S2, unofficial].
3.  **Configuration:** Configure the resampling filter with the following parameters: Transform is Identity, Interpolator is Linear (trilinear) or higher-order (cubic), Size is the calculated output dimensions, OutputSpacing is set to `kTargetSpacing` for all axes (e.g., 1.0F), OutputOrigin and OutputDirection are preserved from the input image, and DefaultPixelValue is 0.0F for empty regions [S2, unofficial][S3].
4.  **Execution:** Execute the resampling filter (e.g., `UpdateLargestPossibleRegion`) to probe the source volume and compute interpolated values for the target grid [S2, unofficial][S3].
5.  **Post-Processing (Optional):** Apply additional operations such as Gaussian smoothing, median filtering, or gradient magnitude generation [S4, unofficial].
6.  **Conversion Back:** Convert the resulting ITK image back to the application's native VolumeData structure using the original spatial geometry [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resample_volume(input_volume, target_spacing):
    # 1. Convert to ITK image
    itk_image = volume_to_itk_image(input_volume)
    
    # 2. Calculate output dimensions
    output_size = [
        std.size_t(resampled_dimension(size, spacing, target_spacing))
        for size, spacing in zip(itk_image.size, itk_image.spacing)
    ]
    
    # 3. Configure filter
    filter = ResampleImageFilter()
    filter.SetInput(itk_image)
    filter.SetTransform(IdentityTransform())
    filter.SetInterpolator(LinearInterpolateImageFunction())
    filter.SetSize(output_size)
    filter.SetOutputSpacing([target_spacing] * 3)
    filter.SetOutputOrigin(itk_image.origin)
    filter.SetOutputDirection(itk_image.direction)
    filter.SetDefaultPixelValue(0.0)
    
    # 4. Execute
    filter.UpdateLargestPossibleRegion()
    
    # 5. Convert back
    return itk_image_to_volume(filter.GetOutput(), input_volume.spatial_geometry)
```

**Interpolation Details:**
*   **Standard Data:** Hardware trilinear interpolation is commonly used for attribute and density lookups due to its efficiency [S1, S3].
*   **Phase Data:** For multi-phase data (e.g., SPH phases), custom interpolation using a maximum operator is required to preserve phase boundaries, as standard linear interpolation fails to reconstruct interfaces precisely [S1].
*   **Quality vs. Artifacts:** Trilinear interpolation is easy to implement but can introduce visual artifacts (e.g., wood-like artifacts) [S4, unofficial]. If quality is crucial, piecewise cubic polynomials are required [S3].

**Parameters:**
*   **kTargetSpacing:** The target isotropic spacing value. A common hardcoded value is 1.0F [S2, unofficial].
*   **Texture Format:** Power-of-two dimensions are preferred for GPU performance [S3].

**Pitfalls:**
*   **Interface Reconstruction:** Resampling struggles with the precise reconstruction of phase interfaces due to discretization errors [S1].
*   **Preprocessing Time:** The method requires evaluating data at every grid point, leading to long preprocessing times (minutes) compared to direct rendering methods (<30ms) [S1].
*   **Memory:** Large grid resolutions can exceed available texture memory [S3].

## Subtopics

### [Area-Averaged Resampling](3142-area-averaged-resampling.md)
Area-Averaged Resampling is a downsampling technique that computes the average color of an input image region mapped to a single output pixel, weighted by the area of overlap between the input pixel grid and the mapped output pixel region. It reduces image resolution by a factor of $2^k$ using a box filter (averaging) over a square neighborhood centered at the scaled source coordinates. It distinguishes itself from nearest-neighbor or integer-stride subsampling by accounting for partial pixel coverage.

### [Volume Resampling to Isotropic Spacing](7482-volume-resampling-to-isotropic-spacing.md)
This method resamples a volume with arbitrary anisotropic voxel spacing into a new volume where all axes share a single, uniform spacing value. It solves the problem of inconsistent physical scales across axes, which complicates downstream operations like filtering or rendering that assume uniform grid metrics. Unlike [Area-Averaged Resampling](3142-area-averaged-resampling.md), which focuses on preserving integrated intensity during downsampling, this method strictly enforces geometric isotropy by calculating output dimensions based on physical extent rather than simple integer division of voxel counts.

## Operating characteristics
*   **Time Complexity:** Preprocessing time is high, proportional to the number of target grid points and the cost of source data evaluation; sources report preprocessing times of several minutes for complex datasets [S1].
*   **Rendering Time:** Significantly faster than direct particle rendering, typically ranging from 82 ms to 293 ms per frame [S1].
*   **Memory Complexity:** Requires storing the full 3D grid, with memory usage scaling with the product of the output dimensions; power-of-two texture constraints may increase memory overhead [S3].
*   **Strengths:** Enables fast volume rendering of complex data; leverages hardware-accelerated texture sampling [S1, S3].
*   **Limitations:** Long preprocessing latency; loss of fine detail at phase interfaces due to discretization; potential visual artifacts from linear interpolation [S1, S3][S4, unofficial].
*   **When to Use:** When rendering speed is prioritized over preprocessing time and when the data can be adequately approximated by a grid structure [S1, S3].
*   **When Not to Use:** When precise reconstruction of sharp phase boundaries is required, or when preprocessing time is constrained [S1].

## Background
Volume resampling was introduced as a standard approach for rendering particle-based data, such as Smoothed Particle Hydrodynamics (SPH) simulations, by converting them into a uniform grid representation [S1]. The method allows the use of efficient direct volume rendering techniques on data that is inherently unstructured [S1]. Early implementations utilized GPU fragment shaders for on-the-fly resampling [S4, unofficial], while later approaches leveraged toolkits like ITK for robust preprocessing [S2, unofficial].

## Sources
[S1] Stochastic Volume Rendering of Multi‐Phase SPH Data, https://doi.org/10.1111/cgf.14121, fetched 2026-10-27
[S2] wpop/qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
[S3] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S4] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
