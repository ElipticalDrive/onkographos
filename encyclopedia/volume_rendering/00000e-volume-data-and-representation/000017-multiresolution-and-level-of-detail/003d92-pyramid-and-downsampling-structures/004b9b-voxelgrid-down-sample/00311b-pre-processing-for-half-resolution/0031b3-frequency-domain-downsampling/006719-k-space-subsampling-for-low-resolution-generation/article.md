# K-space subsampling for low-resolution generation

Also called: k-space cropping, low-resolution volume generation

## Overview
Generates a low-resolution volumetric representation from a high-resolution input by subsampling the central region of its 3D Fourier transform (k-space) and reconstructing via inverse transform [S1, unofficial]. It differs from spatial downsampling by preserving low-frequency structural information while discarding high-frequency details, serving as a frequency-domain alternative to spatial pooling or strided convolution [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A 3D volumetric array `volume` of shape `(D, H, W)` (depth, height, width).
- Output: A 3D volumetric array `reconstructed` of shape `(D//scale, H//scale, W//scale)` normalized to the range `[-1, 1]` [S1, unofficial].

**Data Structures**
- `volume`: Input tensor/array.
- `k_space`: Complex-valued array resulting from the 3D FFT.
- `selected_k_space`: Sub-array of `k_space` containing the central frequencies.
- `reconstructed_image`: Complex-valued array from the inverse FFT.
- `reconstructed`: Final output array.

**Parameters**
- `scale`: Integer downsampling factor (e.g., 2, 4, 8). Determines the reduction ratio [S1, unofficial].
- `central_fraction`: Float, default `0.5`. Fraction of the full k-space dimensions used to calculate the base crop size before dividing by `scale` [S1, unofficial].

**Steps**
1. **Compute 3D FFT and Shift**: Calculate the 3D Fast Fourier Transform of the input volume and shift the zero-frequency component to the center.
   $$ k\_space = \text{fftshift}(\text{fft3}(volume)) $$
   *Note: The source code uses `fft2` on 3D data, which in standard libraries (like NumPy) implies a 2D transform. However, the context is 3D volumes. The implementation must use a 3D FFT (`fft3` or `fftn` with 3 axes) to match the 3D input shape `(deps, rows, cols)` [S1, unofficial].*

2. **Calculate Center Coordinates**: Determine the center indices of the k-space array.
   $$ center\_dep = \lfloor D / 2 \rfloor, \quad center\_row = \lfloor H / 2 \rfloor, \quad center\_col = \lfloor W / 2 \rfloor $$

3. **Determine Crop Sizes**: Calculate the base dimensions for the central crop using `central_fraction`.
   $$ central\_deps = \lfloor 0.5 \times D \rfloor $$
   $$ central\_rows = \lfloor 0.5 \times H \rfloor $$
   $$ central\_cols = \lfloor 0.5 \times W \rfloor $$

4. **Extract Central K-Space Region**: Slice the k-space array to retain only the low-frequency components. The slice bounds are defined by the center coordinates adjusted by the base crop size divided by the `scale` factor.
   $$ start\_dep = center\_dep - \lfloor central\_deps / scale \rfloor $$
   $$ end\_dep = center\_dep + \lfloor central\_deps / scale \rfloor $$
   (Similarly for rows and cols).
   $$ selected\_k\_space = k\_space[start\_dep:end\_dep, start\_row:end\_row, start\_col:end\_col] $$
   *Note: The source code uses `//` (integer division) for the slice bounds. The resulting shape is approximately `(D/scale, H/scale, W/scale)`.*

5. **Reconstruct via Inverse FFT**: Apply the inverse FFT to the cropped k-space data. First, shift the zero-frequency component back to the corners, then compute the inverse transform.
   $$ reconstructed\_image = \text{ifft3}(\text{fftshift}(selected\_k\_space)) $$

6. **Normalize**: Normalize the reconstructed volume to the range `[-1, 1]`.
   - Let $M = \max(|reconstructed\_image|)$ and $m = \min(|reconstructed\_image|)$.
   - If $M \neq 0$ and $M \neq m$:
     $$ reconstructed = 2 \times \frac{|reconstructed\_image| - m}{M - m} - 1 $$
   - Else (if $M=0$ or $M=m$):
     $$ reconstructed = |reconstructed\_image| $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def k_space_subsampling(volume, scale=2, central_fraction=0.5):
    """
    Generates a low-resolution volume from a high-resolution volume by subsampling
    the central region of its 3D Fourier transform.
    
    Args:
        volume: 3D numpy array of shape (D, H, W).
        scale: Integer downsampling factor.
        central_fraction: Fraction of full dimensions for base crop size.
        
    Returns:
        reconstructed: 3D numpy array of shape (D//scale, H//scale, W//scale).
    """
    # 1. Compute 3D FFT and shift
    # Note: fftn computes the N-dimensional discrete Fourier Transform.
    k_space = np.fft.fftshift(np.fft.fftn(volume))
    
    # 2. Calculate center coordinates
    deps, rows, cols = k_space.shape
    center_dep = deps // 2
    center_row = rows // 2
    center_col = cols // 2
    
    # 3. Determine crop sizes
    central_deps = int(central_fraction * deps)
    central_rows = int(central_fraction * rows)
    central_cols = int(central_fraction * cols)
    
    # 4. Extract central k-space region
    # Calculate slice bounds
    dep_start = center_dep - central_deps // scale
    dep_end = center_dep + central_deps // scale
    row_start = center_row - central_rows // scale
    row_end = center_row + central_rows // scale
    col_start = center_col - central_cols // scale
    col_end = center_col + central_cols // scale
    
    selected_k_space = k_space[dep_start:dep_end, row_start:row_end, col_start:col_end]
    
    # 5. Reconstruct via inverse FFT
    # Shift back to corner-centered for ifftn, then inverse transform
    reconstructed_image = np.fft.ifftn(np.fft.ifftshift(selected_k_space))
    
    # 6. Normalize
    abs_image = np.abs(reconstructed_image)
    max_val = np.max(abs_image)
    min_val = np.min(abs_image)
    
    if max_val != 0 and max_val != min_val:
        reconstructed = 2 * (abs_image - min_val) / (max_val - min_val) - 1
    else:
        reconstructed = abs_image
        
    return reconstructed
```

**Edge Cases and Pitfalls**
- **Division by Zero**: If the maximum absolute value of the reconstructed image is 0, or if the maximum equals the minimum, the normalization step is skipped, and the absolute values are returned directly [S1, unofficial].
- **FFT Dimensionality**: The source code snippet mentions `fft2` but operates on 3D data. Implementers must ensure a 3D FFT (`fft3` or `fftn` along all 3 axes) is used to maintain volume integrity [S1, unofficial].
- **Slice Boundaries**: Integer division (`//`) in slice bounds may result in slight asymmetries in the cropped region, especially for odd dimensions. This is consistent with the source implementation [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by the 3D FFT and inverse FFT, which is $O(N \log N)$ where $N$ is the number of voxels in the input volume. The cropping and normalization are linear $O(N_{out})$.
- **Memory Complexity**: Requires storage for the input volume, the full k-space array, and the cropped k-space array. Total memory is roughly $2 \times N_{in} + N_{out}$ complex numbers [S1, unofficial].
- **Strengths**: Fast, non-iterative, and preserves low-frequency structural information. No training required.
- **Limitations**: Loses high-frequency details (sharp edges, fine textures). The output is limited by the resolution defined by `scale`. Not suitable for tasks requiring high-frequency fidelity [S1, unofficial].
- **When to use**: Pre-processing for super-resolution networks, initial coarse estimation, or when computational resources are limited [S1, unofficial].
- **When not to use**: When high-frequency details are critical, or when the input volume has significant high-frequency content that needs to be preserved [S1, unofficial].

## Background
This method is derived from implementations in deep learning-based volume compression and super-resolution pipelines, specifically the `EndtoEndCompression` repository [S1, unofficial]. It leverages the property that most signal energy in natural images/volumes is concentrated in the low-frequency components of the Fourier domain. By cropping the center of k-space, one effectively performs a low-pass filter followed by downsampling [S1, unofficial].

## Sources
[S1] asheibanifard/EndtoEndCompression, https://github.com/asheibanifard/EndtoEndCompression, fetched 2026-10-27 (unofficial)
