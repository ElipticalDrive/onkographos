# Image-Based Volumetric Shadows

Also called: Tóth, Wyman

## Overview
Image-Based Volumetric Shadows is a screen-space technique that computes volumetric shadows using image-based primitives and epipolar geometry [S1, unofficial]. Unlike the parent method [Soft BiF Volumetric Shadows](18810-soft-bif-volumetric-shadows.md) which relies on spatial interpolation of sampled radiance, this approach uses downsampling, interleaved sampling, and bilateral filtering to determine shadowing along view rays [S1, unofficial]. It is distinct from geometric-based methods [Geometrical-Based Volumetric Shadows](18811-geometrical-based-volumetric-shadows.md) by replacing explicit polygon casting or depth peeling with image-space operations [S1, unofficial].

## How it works
The method is a family of screen-space techniques that share the goal of computing volumetric shadows using image-based primitives [S1, unofficial]. Implementations vary by author, but all operate in screen space using depth buffers and epipolar lines [S1, unofficial].

### Inputs and Outputs
- **Inputs**: Depth buffer (Z-buffer), view direction, light direction, image resolution [S1, unofficial].
- **Outputs**: Shadow factor per pixel, often integrated into a single-scattering radiance term [S1, unofficial].

### Data Structures
- **Depth Buffer**: Standard screen-space depth values [S1, unofficial].
- **Downsampled Buffer**: Lower-resolution representation of shadow/scattering state [S1, unofficial].
- **Epipolar Line Data**: Linear arrays of samples along epipolar lines in screen space [S1, unofficial].

### Steps
The general workflow involves placing samples along epipolar lines, detecting depth discontinuities, and applying filtering [S1, unofficial]. Specific implementations differ:

1. **Epipolar Line Construction**: Transform eye rays and radiance rays into a coordinate system where epipolar lines are aligned with screen rows or columns (rectification) [S1, unofficial].
2. **Sample Placement**:
   - *Tóth*: Compute downsamples for each pixel; neighboring pixels share results [S1, unofficial].
   - *Engelhardt*: Linearly search epipolar lines for Z-buffer discontinuities; place samples only after and before these positions [S1, unofficial].
   - *Wyman [36]*: Voxelize image-based shadow casters in epipolar geometry instead of ray marching [S1, unofficial].
3. **Visibility Testing**:
   - *Tóth*: Use interleaved sampling to evaluate if a sample position along the view-ray is in scattering or shadow [S1, unofficial].
   - *Wyman [26]*: Add polygons along the view ray; clamp the segment between the front and farthest polygon, then perform visibility testing against image-based shadows [S1, unofficial].
4. **Filtering**:
   - *Engelhardt*: Apply bilateral filtering along and between epipolar lines to preserve downsamples for single scattering [S1, unofficial].
   - *Ali*: Use bilateral filtering to generate soft shadow maps using multiple image-based algorithms [S1, unofficial].
5. **Incremental Evaluation**:
   - *Baran*: Evaluate radiance incrementally along the same epipolar line using rectified shadow maps [S1, unofficial].
   - *Chen*: Accelerate partial sum tree method by replacing one-dimensional min-max mipmap calculation [S1, unofficial].

### Pseudocode
The following pseudocode represents a generalized implementation based on Tóth and Engelhardt, as no single source provides a complete unified algorithm.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_image_based_volumetric_shadows(depth_buffer, light_dir, view_dir, resolution):
    # Step 1: Rectify coordinates to align epipolar lines with screen rows
    # [G] Transform view and light directions to a coordinate system where
    # epipolar lines are horizontal. Exact transformation not given in sources.
    rectified_depth = depth_buffer  # [G] Placeholder for rectification transform

    # Step 2: Detect Z-buffer discontinuities along epipolar lines
    # [G] Linear search for depth jumps along each row
    discontinuities = []
    for y in range(resolution.height):
        row = rectified_depth[y]
        for x in range(1, resolution.width):
            if abs(row[x] - row[x-1]) > 0.01:  # [G] Threshold not given in sources
                discontinuities.append((y, x))

    # Step 3: Place samples around discontinuities
    # [G] Samples are placed only after and before discontinuity positions
    sample_positions = []
    for y, x in discontinuities:
        sample_positions.append((y, x - 1))
        sample_positions.append((y, x))
        sample_positions.append((y, x + 1))

    # Step 4: Interleaved sampling for shadow/scattering state
    # [G] Evaluate if sample is in shadow or scattering using image-based shadows
    shadow_flags = {}
    for y, x in sample_positions:
        # [G] Interleaved sampling checks visibility along the view ray
        shadow_flags[(y, x)] = True  # [G] Exact visibility test logic not given in sources

    # Step 5: Bilateral filtering to preserve edges
    # [G] Filter along and between epipolar lines
    filtered_shadows = {}
    for y, x in shadow_flags:
        # [G] Standard bilateral filter along epipolar lines
        filtered_shadows[(y, x)] = shadow_flags[(y, x)]  # [G] Exact filter kernel not given in sources

    return filtered_shadows
```

### Parameters
- **Filter Radius**: Not given in the sources.
- **Downsampling Factor**: Not given in the sources.
- **Discontinuity Threshold**: Not given in the sources.

### Edge Cases and Pitfalls
Not given in the sources.

## Operating characteristics
- **Time Complexity**: Depends on the number of samples per pixel and filter radius [G]. Generally faster than ray marching due to screen-space operations [G].
- **Memory Complexity**: Requires depth buffer and intermediate buffers for downsampling and filtering [G].
- **Strengths**: Real-time performance, avoids volumetric grid storage, leverages existing depth buffers [G].
- **Limitations**: Screen-space only (no off-screen shadows), artifacts at depth discontinuities, dependent on depth buffer accuracy [G].
- **When to Use**: Real-time applications where performance is critical and screen-space shadows are sufficient [G].
- **When Not to Use**: When off-screen shadows are required or when high-quality volumetric scattering is needed [G].

## Background
The method was proposed by multiple authors independently:
- **Wyman [26, 36]**: Proposed geometric-based shadows with polygon clamping and later voxelize shadow casters in epipolar geometry [S1, unofficial].
- **Tóth [23]**: Introduced downsampling and interleaved sampling in pixel shaders [S1, unofficial].
- **Engelhardt [24]**: Presented epipolar line sampling with bilateral filtering [S1, unofficial].
- **Ali [35]**: Used bilateral filtering for soft shadow maps [S1, unofficial].
- **Baran [25]**: Proposed incremental radiance evaluation along epipolar lines [S1, unofficial].
- **Chen [27]**: Accelerated the method using min-max mipmaps [S1, unofficial].

## Sources
[S1] A Review On Light Shafts Rendering For Indoor Scenes, https://doi.org/10.5281/zenodo.1125510, fetched 2026-10-26 (unofficial)
