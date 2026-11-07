# FDK Pre-Filtering (Ramp Filter + Cone Weighting)
Also called: CPU FDK Pre-Filter, FDK Filter Application, Ramp Filter Kernel Mapping, applyFDKPreFilter

## Overview
FDK Pre-Filtering applies the Feldkamp-Davis-Kress (FDK) pre-processing step to raw projection data, preparing it for backprojection in explicit cone-beam geometry [S1, unofficial]. It solves the problem of correcting for the divergent ray paths and high-frequency attenuation inherent in cone-beam acquisitions by combining geometric cone weighting with a 1D ramp filter [S2, unofficial]. Unlike parallel-beam Filtered Back Projection (FBP), this method accounts for the distance from the source to the detector and the angle of incidence, ensuring accurate reconstruction for divergent beams [S1, unofficial]. It sets itself apart from [Adaptive Frequency Domain Filtering](968-adaptive-frequency-domain-filtering.md) by using a fixed ramp kernel rather than adaptive frequency masks, and from [Hybrid CPU-GPU FVR Pipeline](7402-hybrid-cpu-gpu-fvr-pipeline.md) by being the explicit CPU-side implementation of this specific pre-filtering stage [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D array of projections `projections` of shape `(P, H, W)` or `(P, W, H)` depending on axis orientation, where `P` is the number of views, `H` is the number of detector rows, and `W` is the number of detector columns. Parameters include `voxelSize` (mm), `pixelSize` (mm), `sdd` (Source-to-Detector Distance in mm), and `axis` (integer indicating the filtering dimension, typically `-1` for the last dimension).
*   **Output:** A 3D array of filtered projections of the same shape as the input.

**Data Structures:**
*   `projections`: `std::vector<float>` or `torch.Tensor` containing raw projection values.
*   `filter_kernel`: `std::vector<double>` or `torch.Tensor` representing the ramp filter in the frequency domain.
*   `row_vector`: `std::vector<std::complex<double>>` used for intermediate FFT storage.

**Numbered Steps:**

1.  **Axis Normalization:** If the `axis` parameter is not `-1`, transpose the `projections` array so that the filtering dimension is always the last one (columns). This ensures consistent processing logic [S1, unofficial].
2.  **Cone Weighting:** For each projection `p` in `0..P-1`:
    *   Calculate the center coordinates for each detector pixel. For column index `u` in `0..W-1` and row index `v` in `0..H-1`:
        $$ a = -(u - 0.5(W-1)) \times \text{pixelSize} $$
        $$ b = (v - 0.5(H-1)) \times \text{pixelSize} $$
    *   Calculate the geometric cone weight `cw`:
        $$ cw = \frac{\text{sdd}}{\sqrt{\text{sdd}^2 + a^2 + b^2}} $$
    *   Apply the weight and normalize by voxel size:
        $$ \text{proj}[u, v] = \frac{\text{proj}[u, v]}{\text{voxelSize}} \times cw $$
    *   This step corrects for the divergent geometry where rays hitting the detector at an angle or further from the center have longer paths and different intensities [S2, unofficial].
3.  **FFT Preparation:** For each row `v` in `0..H-1` of the current projection:
    *   Extract the row data into a temporary vector.
    *   Determine the padded length `padN` for the FFT. The sources provide two strategies:
        *   Strategy A (from [S2, unofficial]): `padN = nextPow2(W)`, where `nextPow2(n)` returns the smallest power of 2 greater than or equal to `n`.
        *   Strategy B (from [S1, unofficial]): `padded = max(64, 1 << (2 * length - 1).bit_length())`.
        *   Zero-pad the row to length `padN`.
4.  **Ramp Filter Construction:** Build the ramp filter kernel of size `padN`.
    *   Initialize a real-valued vector `f` of size `padN` with zeros.
    *   Set `f[0] = 0.25`.
    *   For odd indices `i` from `1` to `padN/2`:
        $$ f[1 + 2i] = \frac{-1.0}{\pi^2 i^2} $$
    *   For odd indices `i` from `padN/2 - 1` down to `1`:
        $$ f[1 + 2i] = \frac{-1.0}{\pi^2 i^2} $$
        *(Note: This logic effectively fills the symmetric negative frequencies based on the odd-indexed harmonic components of the ramp function)* [S2, unofficial].
    *   Convert `f` to a complex vector `fc`.
    *   Perform an in-place FFT on `fc` with sign `-1` (forward transform).
    *   Multiply each element `fc[k]` by `2.0 * real(fc[k])`. This step effectively creates the frequency-domain ramp filter response $|k|$ scaled appropriately [S2, unofficial].
    *   The resulting real parts form the filter kernel applied to the projection rows.
5.  **Frequency Domain Filtering:** For each row `v` in `0..H-1`:
    *   Perform an in-place FFT on the zero-padded row vector with sign `-1`.
    *   Multiply each element of the transformed row by the corresponding element of the pre-computed ramp filter kernel.
    *   Perform an in-place FFT on the result with sign `+1` (inverse transform).
    *   Normalize the result:
        $$ \text{norm} = \frac{1.0}{\text{padN} \times 2.0} $$
    *   Update the original projection row `proj[u, v]` with the real part of the first `W` elements of the inverse FFT result, scaled by `norm`.
6.  **Chunking (Optional Optimization):** If processing large batches, the filtering can be applied in chunks. Calculate `chunk = max(1, _FILTER_CHUNK // (batch * count * padded))`. Apply the filter to slices `lines[:, 0, start : start + chunk]` and concatenate the results [S1, unofficial].
7.  **Axis Restoration:** If the `axis` parameter was not `-1`, transpose the output array back to the original orientation [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import cmath

def next_pow2(n):
    """Returns the smallest power of 2 greater than or equal to n."""
    if n <= 0:
        return 1
    return 1 << (n - 1).bit_length()

def fft_inplace(a, sign):
    """
    Performs an in-place FFT on a list of complex numbers.
    sign: -1 for forward, +1 for inverse.
    """
    N = len(a)
    # Bit-reversal permutation
    j = 0
    for i in range(1, N):
        bit = N >> 1
        while j & bit:
            j ^= bit
            bit >>= 1
        j ^= bit
        if i < j:
            a[i], a[j] = a[j], a[i]
    
    length = 2
    while length <= N:
        angle = sign * 2.0 * math.pi / length
        wlen = cmath.exp(complex(0, angle))
        for i in range(0, N, length):
            w = complex(1, 0)
            for j in range(length // 2):
                u = a[i + j]
                v = a[i + j + length // 2] * w
                a[i + j] = u + v
                a[i + j + length // 2] = u - v
                w *= wlen
        length *= 2

def build_ramp_filter(size):
    """
    Builds the ramp filter kernel in the frequency domain.
    Returns a list of complex numbers.
    """
    f = [0.0] * size
    f[0] = 0.25
    
    # Fill odd indices based on the formula -1 / (pi^2 * i^2)
    # The source logic collects odd indices 1..size/2 and size/2-1..1
    # This effectively populates the symmetric parts of the filter
    half = size // 2
    for i in range(1, half + 1, 2):
        if i < size:
            f[i] = -1.0 / (math.pi * math.pi * i * i)
    for i in range(half - 1, 0, -2):
        if i < size:
            f[i] = -1.0 / (math.pi * math.pi * i * i)
            
    # Convert to complex and FFT
    fc = [complex(x, 0) for x in f]
    fft_inplace(fc, -1)
    
    # Multiply by 2 * real part
    for i in range(size):
        fc[i] = complex(2.0 * fc[i].real, 0)
        
    return fc

def apply_fdk_pre_filter(projections, P, W, H, voxel_size, sdd, pixel_size):
    """
    Applies FDK pre-filtering (cone weighting + ramp filter) to projections.
    projections: list of lists of lists (P, H, W) or flattened equivalent.
    Returns filtered projections.
    """
    # 1. Cone Weighting
    filtered = [[0.0] * W for _ in range(H)]
    
    for p in range(P):
        # Get current projection slice
        # Assuming projections is a flat list or 3D array, accessing element p
        current_proj = projections[p] 
        
        for u in range(W):
            a = -(u - 0.5 * (W - 1)) * pixel_size
            for v in range(H):
                b = (v - 0.5 * (H - 1)) * pixel_size
                cw = sdd / math.sqrt(sdd * sdd + a * a + b * b)
                # Apply weight and normalize
                val = current_proj[u * H + v] # Assuming row-major or specific layout
                filtered[v][u] = (val / voxel_size) * cw
                
        # 2. FFT Filtering per row
        pad_n = next_pow2(W)
        ramp_filter = build_ramp_filter(pad_n)
        
        for v in range(H):
            # Extract row and pad
            row = [complex(0, 0)] * pad_n
            for u in range(W):
                row[u] = complex(filtered[v][u], 0)
            
            # Forward FFT
            fft_inplace(row, -1)
            
            # Multiply by filter
            for u in range(pad_n):
                row[u] = row[u] * ramp_filter[u]
                
            # Inverse FFT
            fft_inplace(row, 1)
            
            # Normalize and store back
            norm = 1.0 / (pad_n * 2.0)
            for u in range(W):
                filtered[v][u] = row[u].real * norm
                
    return filtered
```

**Parameters:**
*   `P`: Number of projections (views). Typical values: 360–1000 [S2, unofficial].
*   `W`: Detector width in pixels.
*   `H`: Detector height in pixels.
*   `voxelSize`: Physical size of a voxel in the output volume (mm) [S2, unofficial].
*   `sdd`: Source-to-Detector Distance (mm) [S2, unofficial].
*   `pixelSize`: Physical size of a detector pixel (mm) [S2, unofficial].
*   `_FILTER_CHUNK`: Chunk size for memory optimization. Default not given, calculated as `max(1, _FILTER_CHUNK // (batch * count * padded))` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The FFT implementation uses `double` precision for intermediate calculations to maintain accuracy, especially for the ramp filter construction and multiplication [S2, unofficial].
*   **Boundary Conditions:** The zero-padding to the next power of 2 handles the circular convolution artifact of the FFT, assuming the ramp filter is correctly constructed for the padded length [S2, unofficial].
*   **Degenerate Input:** If `W` is 0 or `H` is 0, the loops will not execute, returning an empty result. If `sdd` is 0, division by zero occurs in the cone weight calculation [S2, unofficial].

## Subtopics

### [FDK Geometry Weight Calculation](8285-fdk-geometry-weight-calculation.md)
Computes the geometric weighting factors required for the backprojection step of the Feldkamp-Davis-Kress (FDK) algorithm in explicit cone-beam geometry. It solves the problem of correctly weighting each ray's contribution based on the angle between the ray and the detector panel normal, as well as the distance from the source to the detector. It sets itself apart from parallel-beam FBP by accounting for the divergent cone-beam geometry through a `|dS · ν| / D'` weighting scheme, which simplifies to Feldkamp's cosine weight for circular orbits and upright panels.

## Operating characteristics
*   **Time Complexity:** $O(P \cdot H \cdot W \log W)$ due to the 1D FFT performed on each of the $P \times H$ rows of length $W$ [S2, unofficial].
*   **Memory Complexity:** $O(P \cdot H \cdot W)$ for the input and output projections, plus $O(W)$ for the temporary row vectors and filter kernel [S2, unofficial].
*   **Measured Performance:** On a system with 360 views and 128 detector rows, the total work-items processed are $360 \times 128 = 46,080$ [S2, unofficial]. The CPU implementation validates against a Python reference with MSE $< 10^{-2}$ [S2, unofficial].
*   **Strengths:** Explicitly handles cone-beam geometry, providing accurate pre-filtering for FDK reconstruction. The FFT-based approach is efficient for large detector widths.
*   **Limitations:** Requires padding to power-of-2 lengths, which may increase memory usage slightly. The fixed ramp filter does not adapt to noise characteristics like [Adaptive Frequency Domain Filtering](968-adaptive-frequency-domain-filtering.md) [S2, unofficial].
*   **When to Use:** When performing explicit cone-beam reconstruction using the FDK algorithm and a CPU-based pre-filtering pipeline is required or preferred for validation [S2, unofficial].
*   **When Not to Use:** When using a fully GPU-accelerated pipeline that integrates filtering and backprojection, or when adaptive filtering is needed for noisy data [S2, unofficial].

## Background
The FDK algorithm was introduced by Feldkamp, Davis, and Kress in 1984 for cone-beam CT reconstruction [S1, unofficial]. The pre-filtering step described here is a standard component of the FDK method, adapting the 1D ramp filter used in parallel-beam FBP to account for the divergent geometry of cone-beam scans [S1, unofficial]. The specific implementation details provided in the sources are from unofficial repositories ([S1, unofficial], [S2, unofficial]) that implement and validate this step for CPU and GPU pipelines, ensuring compatibility with standard HDF5-based CT datasets [S2, unofficial].

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S2] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
