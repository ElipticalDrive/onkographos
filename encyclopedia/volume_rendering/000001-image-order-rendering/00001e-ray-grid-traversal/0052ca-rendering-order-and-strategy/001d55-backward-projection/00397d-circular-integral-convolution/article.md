# Circular Integral Convolution

Also called: 1/r Filter Sampling, Back-projection / Circular Integral Convolution, Circular Integral Method, Convolution Filter Construction, Simple Back-Projection

## Overview
Circular Integral Convolution generates a blurred image from a scan by integrating ray values across all angles for each point, effectively filtering the true image with a $1/r$-weighted kernel [S1, unofficial]. It serves as a pre-processing step for deconvolution-based reconstruction rather than a full reconstruction method itself [S1, unofficial].

## How it works
**Inputs:**
- `scan_data`: 1D array of ray values, size `angles * rays` [S1, unofficial].
- `angles`: Number of projection angles [S1, unofficial].
- `rays`: Number of rays per angle [S1, unofficial].
- `width`, `height`: Output image dimensions [S1, unofficial].
- `x_overscan`, `y_overscan`: Margin pixels added to handle non-zero filter values outside original image bounds [S1, unofficial].

**Outputs:**
- `output_image`: 2D grid of pixel values [S1, unofficial].

**Data Structures:**
- Scan data is indexed as `scan_data[angle_num * rays + ray_index]` [S1, unofficial].
- Output image is a 2D array indexed by `(x, y)` where coordinates range from `-1` to `1` (normalized) [S1, unofficial].

**Parameters:**
- `ray_offset`: Constant `sqrt(2)` [S1, unofficial].
- `ray_step_inv`: Calculated as `(rays - 1) / (2 * sqrt(2))` [S1, unofficial].
- `fudge`: Calculated as `sqrt(x_step^2 + y_step^2) / 4.0` to handle singularity at origin [S1, unofficial].
- Typical values: Not given in the sources.

**Steps:**
1. **Initialize Output Grid:** For each pixel `(x, y)` in the output grid (coordinates normalized to -1..1, including overscan margins):
   a. Initialize `total = 0.0` [S1, unofficial].
   b. For `angle_num` from `0` to `angles - 1`:
      i. Calculate `angle = angle_num * (PI / angles)` [S1, unofficial].
      ii. Calculate normal vector `normal = (-sin(angle), cos(angle))` [S1, unofficial].
      iii. Calculate signed distance `dist = normal.x * x + normal.y * y` [S1, unofficial].
      iv. Convert distance to floating-point ray index: `ray_num = (dist + ray_offset) * ray_step_inv` [S1, unofficial].
      v. Determine integer ray indices `r0 = floor(ray_num)` and `r1 = r0 + 1` [S1, unofficial].
      vi. Calculate interpolation fraction `fract = ray_num - r0` [S1, unofficial].
      vii. If `0 <= ray_num < rays`:
          - Retrieve `val0 = scan_data[angle_num * rays + r0]` (if `r0` is valid) [S1, unofficial].
          - Retrieve `val1 = scan_data[angle_num * rays + r1]` (if `r1` is valid) [S1, unofficial].
          - Perform linear interpolation: `interpolated = (1 - fract) * val0 + fract * val1` [S1, unofficial].
          - Add `interpolated` to `total` [S1, unofficial].
   c. Normalize: `output_image[x][y] = total / angles` [S1, unofficial].

2. **Construct Convolution Filter Weights:**
   a. Calculate pixel step sizes: `x_step = 2.0 / width`, `y_step = 2.0 / height` [S1, unofficial].
   b. Calculate fudge factor: `fudge = sqrt(x_step^2 + y_step^2) / 4.0` [S1, unofficial].
   c. For each pixel `(x, y)` in the grid (centered at 0,0):
      i. Calculate radius `r = sqrt(x^2 + y^2)` [S1, unofficial].
      ii. Clamp `r` to be at least `fudge` to avoid division by zero [S1, unofficial].
      iii. Calculate filter weight: `weight = (x_step * y_step) / (r * 2 * PI)` [S1, unofficial].
      iv. Store `weight` in the convolution filter kernel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def circular_integral_convolution(scan_data, angles, rays, width, height, x_overscan, y_overscan):
    output_image = [[0.0] * (width + 2 * x_overscan) for _ in range(height + 2 * y_overscan)]
    ray_offset = sqrt(2)
    ray_step_inv = (rays - 1) / (2 * sqrt(2))
    x_step = 2.0 / width
    y_step = 2.0 / height
    fudge = sqrt(x_step**2 + y_step**2) / 4.0

    for y in range(height + 2 * y_overscan):
        for x in range(width + 2 * x_overscan):
            px = (x - x_overscan) * x_step
            py = (y - y_overscan) * y_step
            total = 0.0
            for angle_num in range(angles):
                angle = angle_num * (PI / angles)
                normal_x = -sin(angle)
                normal_y = cos(angle)
                dist = normal_x * px + normal_y * py
                ray_num = (dist + ray_offset) * ray_step_inv
                r0 = floor(ray_num)
                r1 = r0 + 1
                fract = ray_num - r0
                if 0 <= ray_num < rays:
                    val0 = scan_data[angle_num * rays + r0] if 0 <= r0 < rays else 0.0
                    val1 = scan_data[angle_num * rays + r1] if 0 <= r1 < rays else 0.0
                    interpolated = (1 - fract) * val0 + fract * val1
                    total += interpolated
            output_image[y][x] = total / angles

    filter_kernel = [[0.0] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            px = (x - width / 2) * x_step
            py = (y - height / 2) * y_step
            r = sqrt(px**2 + py**2)
            r = max(r, fudge)
            weight = (x_step * y_step) / (r * 2 * PI)
            filter_kernel[y][x] = weight
    return output_image, filter_kernel
```

**Edge Cases and Pitfalls:**
- **Singularity at Origin:** The $1/r$ filter has a singularity at `r=0`. This is handled by clamping the radius to a minimum `fudge` value, which is half the pixel diagonal [S1, unofficial].
- **Boundary Conditions:** Ray indices must be checked against `0` and `rays`. Indices outside this range are skipped or clamped to valid bounds [S1, unofficial].
- **Filter Normalization:** The filter is not normalized to ensure the integral inside the unit disc is precisely 1.0; instead, it relies on the grid being large enough that the error remains within 0.5% [S1, unofficial].
- **Overscan:** Overscan margins (`x_overscan`, `y_overscan`) are required to handle non-zero filter values outside the original image bounds [S1, unofficial].

## Subtopics

### [FFT-based Deconvolution Reconstruction](8509-fft-based-deconvolution-reconstruction.md)
Uses FFT to invert the convolution process inherent in back-projection, reducing alignment and blurring errors compared to simple back-projection. Choose this when high-fidelity reconstruction is needed after the initial blur.

## Operating characteristics
- **Time Complexity:** $O(A \cdot W \cdot H)$, where $A$ is the number of angles, and $W, H$ are image dimensions. Each pixel requires integration over all angles [S1, unofficial].
- **Memory Complexity:** $O(A \cdot R + W \cdot H)$, for storing the scan data and the output image/filter [S1, unofficial].
- **Performance:** Baseline RMS error for reconstructing an empty image against a test image is between 71.0 and 72.0. The RMS error of the blurry reconstruction against the original test image is between 59.0 and 61.0 [S1, unofficial].
- **Strengths:** Simple to implement; provides a clear blurred intermediate state for deconvolution [S1, unofficial].
- **Limitations:** Not a full reconstruction; suffers from blurring and alignment errors if not followed by deconvolution. Error does not asymptotically head to zero without sufficient rays/angles and proper deconvolution [S1, unofficial].
- **When to Use:** As a pre-processing step for FFT-based deconvolution reconstruction [S1, unofficial].
- **When Not to Use:** When a direct, sharp reconstruction is required without further processing [S1, unofficial].

## Background
The method is documented in the "rediscovering-tomography" project by Simon Frankau [S1, unofficial]. It was developed as part of an investigation into back-projection techniques and their suitability for deconvolution-based reconstruction. The approach was identified as an "interesting dead end" for direct reconstruction due to blurring but proved useful as a basis for understanding the convolution filter required for FFT-based deconvolution [S1, unofficial].

## Sources
[S1] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
