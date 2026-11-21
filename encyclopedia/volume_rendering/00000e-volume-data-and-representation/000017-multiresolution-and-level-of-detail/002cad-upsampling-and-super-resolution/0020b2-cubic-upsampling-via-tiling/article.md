# Cubic Upsampling via Tiling
## Overview
Cubic Upsampling via Tiling increases the resolution of volumetric data by replicating existing voxels into larger blocks [S1, unofficial]. It generates a higher-resolution grid from a lower-resolution source by strictly copying values, which preserves sharp discontinuities without interpolation blurring [G]. It differs from neural super-resolution methods [Deep Learning-Based Super-Resolution](12769-deep-learning-based-super-resolution.md) by requiring no training data, and from [3D Fast Bilateral Solver](6478-3d-fast-bilateral-solver.md) by ignoring edge-awareness constraints [G].

## How it works
**Inputs:**
- `input_values`: A 1D array of values representing the low-resolution volume.
- `N_prev`: The resolution of the input volume along each axis (assumed cubic, i.e., $N_{prev} \times N_{prev} \times N_{prev}$) [S1, unofficial].
- `N_new`: The target resolution of the output volume [S1, unofficial].

**Outputs:**
- `output_values`: A 1D array of values representing the high-resolution volume [S1, unofficial].

**Data Structures:**
- The input is conceptually treated as a 3D cube of shape $(N_{prev}, N_{prev}, N_{prev})$ [S1, unofficial].
- The output is a 3D cube of shape $(N_{new}, N_{new}, N_{new})$ [S1, unofficial].
- Memory layout follows row-major (C-style) ordering for both 1D arrays [G].

**Parameters:**
- `scale_factor` ($s$): An integer $> 1$, calculated as $s = N_{new} / N_{prev}$ [S1, unofficial]. Typical values are 2, 4, or 8 [G].

**Steps:**
1. Calculate the integer scale factor $s = N_{new} / N_{prev}$ [S1, unofficial].
2. Reshape the 1D `input_values` array into a 3D array `cube` of shape $(N_{prev}, N_{prev}, N_{prev})$ [S1, unofficial].
3. Tile `cube` along the Z-axis (dimension 2) by factor $s$, creating an intermediate array of shape $(N_{prev}, N_{prev}, N_{prev} \times s)$ [S1, unofficial].
4. Tile the result along the Y-axis (dimension 1) by factor $s$, creating an intermediate array of shape $(N_{prev}, N_{prev} \times s, N_{prev} \times s)$ [S1, unofficial].
5. Tile the result along the X-axis (dimension 0) by factor $s$, creating the final 3D array of shape $(N_{prev} \times s, N_{prev} \times s, N_{prev} \times s)$ [S1, unofficial].
6. Reshape the final 3D array back into a 1D `output_values` array [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_upsample_via_tiling(input_values, N_prev, N_new):
    if N_new % N_prev != 0:
        raise ValueError("N_new must be an integer multiple of N_prev")
    s = N_new // N_prev

    # Step 2: Reshape 1D to 3D (N_prev, N_prev, N_prev)
    cube = [[[input_values[i * N_prev * N_prev + j * N_prev + k] for k in range(N_prev)] for j in range(N_prev)] for i in range(N_prev)]

    # Step 3: Tile along Z-axis (dim 2) -> (N_prev, N_prev, N_prev * s)
    tiled_z = [[[cube[i][j][k] for _ in range(s)] for k in range(N_prev)] for j in range(N_prev) for i in range(N_prev)]

    # Step 4: Tile along Y-axis (dim 1) -> (N_prev, N_prev * s, N_prev * s)
    tiled_y = []
    for i in range(N_prev):
        row_y = []
        for _ in range(s):
            row_y.append(tiled_z[i])
        tiled_y.append(row_y)

    # Step 5: Tile along X-axis (dim 0) -> (N_prev * s, N_prev * s, N_prev * s)
    tiled_x = []
    for _ in range(s):
        tiled_x.append(tiled_y)

    # Step 6: Reshape 3D back to 1D
    output_values = [tiled_x[i][j][k] for i in range(N_new) for j in range(N_new) for k in range(N_new)]

    return output_values
```

**Edge Cases and Pitfalls:**
- **Non-integer Scale:** If $N_{new}$ is not divisible by $N_{prev}$, the method fails [S1, unofficial].
- **Memory Usage:** The output size grows by $s^3$, which can exceed available RAM for large $s$ [G].
- **Data Type:** The method preserves the input data type without normalization or clamping [G].

## Subtopics

### Ambient Occlusion UpSampler
Not given in the sources.

## Operating characteristics
- **Time Complexity:** $O(N_{new}^3)$, as every output voxel must be written [G]. The operation is memory-bandwidth bound [G].
- **Memory Complexity:** $O(N_{new}^3)$ for the output array [G].
- **Performance:** Extremely fast compared to interpolation or neural methods, as it involves only memory copying and reshaping [G].
- **Strengths:** Preserves sharp edges and discrete values; no blurring; deterministic; no training required [G].
- **Limitations:** Introduces blocky artifacts if the scale factor is large; does not recover high-frequency details; requires integer scale factors [G].
- **When to use:** Pre-processing for visualization, generating initial guesses for iterative solvers, or when sharp boundaries are critical [G].
- **When not to use:** When smooth transitions are required, or when the scale factor is non-integer [G].

## Background
This method is a fundamental technique in computer graphics for resizing volumetric data without learning-based approaches [G]. It is often used as a baseline or preprocessing step in volume rendering pipelines [G]. The core algorithmic steps are documented in the DIST-Renderer project [S1, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
