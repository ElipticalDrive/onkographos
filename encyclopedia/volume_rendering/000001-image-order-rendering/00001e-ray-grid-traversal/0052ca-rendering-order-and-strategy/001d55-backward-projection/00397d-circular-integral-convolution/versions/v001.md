# Circular Integral Convolution

Also called: 1/r Filter Sampling, Back-projection / Circular Integral Convolution, Circular Integral Method, Convolution Filter Construction, Simple Back-Projection

## Overview
Circular Integral Convolution generates a blurred image from a scan by integrating ray values across all angles for each point, effectively filtering the true image with a $1/r$-weighted kernel [S1, unofficial]. It serves as a pre-processing step for deconvolution-based reconstruction rather than a full reconstruction method itself [S1, unofficial]. It differs from [GPU Backprojection](5843-gpu-backprojection.md) by explicitly constructing the convolution filter weights for subsequent deconvolution, and from [Ray Segment Projection to Image](7631-ray-segment-projection-to-image.md) by operating in the back-projection domain to create a spatially blurred result rather than projecting segments directly to image space.

## How it works
**Inputs:**
- `scan_data`: A 1D array of ray values, size `angles * rays`.
- `angles`: Number of projection angles.
- `rays`: Number of rays per angle.
- `width`, `height`: Output image dimensions.
- `x_overscan`, `y_overscan`: Margin pixels for filter support.

**Outputs:**
- `output_image`: A 2D grid of pixel values.

**Data Structures:**
- The scan data is indexed as `scan_data[angle_num * rays + ray_index]`.
- The output image is a 2D array indexed by `(x, y)` where coordinates range from `-1` to `1` (normalized).

**Parameters:**
- `ray_offset`: Constant `sqrt(2)`.
- `ray_step_inv`: Calculated as `(rays - 1) / (2 * sqrt(2))`.
- `fudge`: Calculated as `sqrt(x_step^2 + y_step^2) / 4.0` to handle singularity at origin.

**Steps:**

1.  **Initialize Output Grid:**
    For each pixel `(x, y)` in the output grid (coordinates normalized to -1..1, including overscan margins):
    a.  Initialize `total = 0.0`.
    b.  For `angle_num` from `0` to `angles - 1`:
        i.  Calculate `angle = angle_num * (PI / angles)`.
        ii. Calculate normal vector `normal = (-sin(angle), cos(angle))`.
        iii. Calculate signed distance `dist = normal.x * x + normal.y * y`.
        iv. Convert distance to floating-point ray index: `ray_num = (dist + ray_offset) * ray_step_inv`.
        v.  Determine integer ray indices `r0 = floor(ray_num)` and `r1 = r0 + 1`.
        vi. Calculate interpolation fraction `fract = ray_num - r0`.
        vii. If `0 <= ray_num < rays`:
            -   Retrieve `val0 = scan_data[angle_num * rays + r0]` (if `r0` is valid).
            -   Retrieve `val1 = scan_data[angle_num * rays + r1]` (if `r1` is valid).
            -   Perform linear interpolation: `interpolated = (1 - fract) * val0 + fract * val1`.
            -   Add `interpolated` to `total`.
            -   [G] If `r0` or `r1` are out of bounds, treat the missing value as 0.0.
    c.  Normalize: `output_image[x][y] = total / angles`.

2.  **Construct Convolution Filter Weights:**
    a.  Calculate pixel step sizes: `x_step = 2.0 / width`, `y_step = 2.0 / height`.
    b.  Calculate fudge factor: `fudge = sqrt(x_step^2 + y_step^2) / 4.0`.
    c.  For each pixel `(x, y)` in the grid (centered at 0,0):
        i.  Calculate radius `r = sqrt(x^2 + y^2)`.
        ii. Clamp `r` to be at least `fudge` to avoid division by zero.
        iii. Calculate filter weight: `weight = (x_step * y_step) / (r * 2 * PI)`.
        iv. Store `weight` in the convolution filter kernel.

**Edge Cases and Pitfalls:**
-   **Singularity at Origin:** The $1/r$ filter has a singularity at `r=0`. This is handled by clamping the radius to a minimum `fudge` value, which is half the pixel diagonal [S1, unofficial].
-   **Boundary Conditions:** Ray indices must be checked against `0` and `rays`. Values outside this range are treated as zero [S1, unofficial].
-   **Filter Normalization:** The filter is not normalized to ensure the integral inside the unit disc is precisely 1.0; instead, it relies on the grid being large enough that the error remains within 0.5% [S1, unofficial].
-   **Overscan:** Overscan margins (`x_overscan`, `y_overscan`) are required to handle non-zero filter values outside the original image bounds [S1, unofficial].

## Subtopics

### [FFT-based Deconvolution Reconstruction](8509-fft-based-deconvolution-reconstruction.md)
Uses FFT to invert the convolution process inherent in back-projection, reducing alignment and blurring errors compared to simple back-projection. Choose this when high-fidelity reconstruction is needed after the initial blur.

## Operating characteristics
-   **Time Complexity:** $O(A \cdot W \cdot H)$, where $A$ is the number of angles, and $W, H$ are image dimensions. Each pixel requires integration over all angles.
-   **Memory Complexity:** $O(A \cdot R + W \cdot H)$, for storing the scan data and the output image/filter.
-   **Performance:** Baseline RMS error for reconstructing an empty image against a test image is between 71.0 and 72.0. The RMS error of the blurry reconstruction against the original test image is between 59.0 and 61.0 [S1, unofficial].
-   **Strengths:** Simple to implement; provides a clear blurred intermediate state for deconvolution.
-   **Limitations:** Not a full reconstruction; suffers from blurring and alignment errors if not followed by deconvolution. Error does not asymptotically head to zero without sufficient rays/angles and proper deconvolution [S1, unofficial].
-   **When to Use:** As a pre-processing step for FFT-based deconvolution reconstruction.
-   **When Not to Use:** When a direct, sharp reconstruction is required without further processing.

## Background
The method originates from experimental tomography reconstruction efforts, specifically documented in the "rediscovering-tomography" project by Simon Frankau [S1, unofficial]. It was developed as part of an investigation into back-projection techniques and their suitability for deconvolution-based reconstruction. The approach was identified as an "interesting dead end" for direct reconstruction due to blurring but proved useful as a basis for understanding the convolution filter required for FFT-based deconvolution.

## Sources
[S1] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
