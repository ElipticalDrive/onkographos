# Cubic Upsampling via Tiling


## Overview
Cubic Upsampling via Tiling is a deterministic, non-learning-based method for increasing the resolution of volumetric data by replicating existing voxels into larger blocks. It solves the problem of generating a higher-resolution grid from a lower-resolution source without introducing interpolation artifacts or blurring, effectively performing nearest-neighbor expansion in 3D space. Unlike bilinear or tri-linear interpolation which smooths boundaries, this method preserves sharp discontinuities in the data by strictly copying values. It differs from neural super-resolution methods [3D Super-Resolution (SR) Module](6577-3d-super-resolution-sr-module.md) by requiring no training data or inference time, and from [3D Fast Bilateral Solver](6478-3d-fast-bilateral-solver.md) by ignoring edge-awareness constraints in favor of computational simplicity.

## How it works
**Inputs:**
- `input_values`: A 1D array of floating-point or integer values representing the low-resolution volume.
- `N_prev`: The resolution of the input volume along each axis (assumed cubic, i.e., $N_{prev} \times N_{prev} \times N_{prev}$).

**Outputs:**
- `output_values`: A 1D array of values representing the high-resolution volume.
- `N_new`: The resolution of the output volume, where $N_{new} = s \times N_{prev}$ and $s$ is the integer scale factor.

**Data Structures:**
- The input is conceptually treated as a 3D cube of shape $(N_{prev}, N_{prev}, N_{prev})$.
- The output is a 3D cube of shape $(N_{new}, N_{new}, N_{new})$.
- Memory layout is row-major (C-style) for both input and output 1D arrays.

**Parameters:**
- `scale_factor` ($s$): An integer $> 1$. Typical values are 2, 4, or 8. Must be derived as $N_{new} / N_{prev}$.

**Steps:**
1. Calculate the integer scale factor $s = N_{new} / N_{prev}$.
2. Reshape the 1D `input_values` array into a 3D array `cube` of shape $(N_{prev}, N_{prev}, N_{prev})$.
3. Tile `cube` along the Z-axis (dimension 2) by factor $s$. This creates an intermediate array of shape $(N_{prev}, N_{prev}, N_{prev} \times s)$.
4. Tile the result along the Y-axis (dimension 1) by factor $s$. This creates an intermediate array of shape $(N_{prev}, N_{prev} \times s, N_{prev} \times s)$.
5. Tile the result along the X-axis (dimension 0) by factor $s$. This creates the final 3D array of shape $(N_{prev} \times s, N_{prev} \times s, N_{prev} \times s)$.
6. Reshape the final 3D array back into a 1D `output_values` array.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_upsample_via_tiling(input_values, N_prev, N_new):
    """
    Upsamples a 1D volume array using tiling.
    
    Args:
        input_values (list or array): 1D array of size N_prev^3.
        N_prev (int): Resolution of the input volume (assumed cubic).
        N_new (int): Resolution of the output volume (assumed cubic).
        
    Returns:
        list or array: 1D array of size N_new^3.
    """
    # Validate inputs
    if N_new % N_prev != 0:
        raise ValueError("N_new must be an integer multiple of N_prev")
    
    s = N_new // N_prev  # Scale factor
    
    # Step 2: Reshape to 3D
    # Assuming input_values is flat and represents a cube N_prev x N_prev x N_prev
    cube = reshape_1d_to_3d(input_values, N_prev, N_prev, N_prev)
    
    # Step 3: Tile along Z-axis (dim 2)
    # Repeat each slice 's' times along the last dimension
    tiled_z = tile_along_axis(cube, axis=2, repeats=s)
    
    # Step 4: Tile along Y-axis (dim 1)
    # Repeat the result 's' times along the middle dimension
    tiled_y = tile_along_axis(tiled_z, axis=1, repeats=s)
    
    # Step 5: Tile along X-axis (dim 0)
    # Repeat the result 's' times along the first dimension
    tiled_x = tile_along_axis(tiled_y, axis=0, repeats=s)
    
    # Step 6: Reshape back to 1D
    output_values = reshape_3d_to_1d(tiled_x)
    
    return output_values

def reshape_1d_to_3d(data, x, y, z):
    """Reshapes 1D data to 3D (x, y, z)."""
    # Implementation detail: standard array reshape operation
    return data.reshape((x, y, z))

def reshape_3d_to_1d(data):
    """Reshapes 3D data to 1D."""
    return data.flatten()

def tile_along_axis(data, axis, repeats):
    """Tiles the array along a specific axis by repeating it 'repeats' times."""
    # Implementation detail: standard numpy-like tile operation
    # np.tile(data, (1, 1, repeats)) for axis=2, etc.
    # Since we cannot call numpy directly in pure python pseudocode without defining it,
    # we describe the logic:
    # Create a new array with shape data.shape[axis] * repeats for the target axis.
    # Copy data into the new array by repeating the source blocks.
    if axis == 0:
        # Repeat along first dimension
        return np.tile(data, (repeats, 1, 1))
    elif axis == 1:
        # Repeat along second dimension
        return np.tile(data, (1, repeats, 1))
    elif axis == 2:
        # Repeat along third dimension
        return np.tile(data, (1, 1, repeats))
    else:
        raise ValueError("Axis must be 0, 1, or 2")
```
*Note: The pseudocode above uses `np.tile` for clarity of intent. In a strict implementation, `tile_along_axis` would involve nested loops copying memory blocks.*

**Edge Cases and Pitfalls:**
- **Non-integer Scale:** If $N_{new}$ is not divisible by $N_{prev}$, the method fails. The sources specify an integer scale factor [S1, unofficial].
- **Memory Usage:** The output size grows by $s^3$. For large $s$, this can exceed available RAM.
- **Data Type:** The method preserves the data type of the input. No normalization or clamping is performed.

## Subtopics

### Ambient Occlusion UpSampler
A specialized variant for upsampling ambient occlusion (AO) data in real-time rendering. It extracts `BentNormal` and `SpecularOcclusion` components from a lower-resolution AO texture (`AoResolution`) to a higher-resolution target (`Resolution`).
- **Definition:** Upsamples AO data using UV coordinate mapping [S2, unofficial].
- **Steps:**
    1. Calculate high-resolution UV coordinates: $uv = \left(\frac{id.x + 0.5}{Resolution.x}, \frac{id.y + 0.5}{Resolution.y}\right)$ [S2, unofficial].
    2. Calculate UV step size based on AO resolution: $uvstep = \left(\frac{1.0}{AoResolution.x}, \frac{1.0}{AoResolution.y}\right) \times direction$ [S2, unofficial].
    3. Sample AO data from the low-res texture using an `UpSampleAmbientOcclusion` function [S2, unofficial].
    4. Write `BentNormal` and `SpecularOcclusion` to the high-res target [S2, unofficial].
- **When to choose:** When working with deferred shading pipelines where AO is precomputed at lower resolution to save memory/bandwidth, and directional information (BentNormal) is required at full resolution [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{new}^3)$, as every output voxel must be written. The tiling operation is memory-bandwidth bound.
- **Memory Complexity:** $O(N_{new}^3)$ for the output array. Intermediate tiled arrays may temporarily double or triple memory usage depending on implementation.
- **Performance:** Extremely fast compared to interpolation or neural methods, as it involves only memory copying and reshaping. No arithmetic operations per voxel.
- **Strengths:** Preserves sharp edges and discrete values; no blurring; deterministic; no training required.
- **Limitations:** Introduces "blocky" artifacts (aliasing) if the scale factor is large; does not recover high-frequency details lost in the original low-res volume; requires integer scale factors.
- **When to use:** Pre-processing for visualization, generating initial guesses for iterative solvers, or when sharp boundaries are critical and interpolation is undesirable.
- **When not to use:** When smooth transitions are required, or when the scale factor is non-integer.

## Background
This method is a fundamental technique in computer graphics for resizing volumetric data without learning-based approaches. It is often used as a baseline or preprocessing step in volume rendering pipelines. The specific implementation details described here are derived from the DIST-Renderer project [S1, unofficial]. The Ambient Occlusion variant is used in real-time rendering contexts, such as the SphereTracing project [S2, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] SphereTracing, https://github.com/nicoell/SphereTracing, fetched 2026-10-26 (unofficial)
