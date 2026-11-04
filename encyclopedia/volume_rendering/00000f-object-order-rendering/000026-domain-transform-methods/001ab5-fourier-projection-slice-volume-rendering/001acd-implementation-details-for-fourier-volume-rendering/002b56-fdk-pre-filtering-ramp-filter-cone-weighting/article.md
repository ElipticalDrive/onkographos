# FDK Pre-Filtering (Ramp Filter + Cone Weighting)
Also called: CPU FDK Pre-Filter, Ramp Filter Kernel Mapping, applyFDKPreFilter

## Overview
This method implements the pre-filtering stage of the Feldkamp-Davis-Kress (FDK) algorithm on the CPU, preparing 2D projection data for subsequent backprojection. It solves the problem of correcting for the cone-beam geometry and high-frequency attenuation inherent in filtered backprojection by applying a ramp filter and cone weighting to each projection slice. It is distinguished from its sibling [Adaptive Frequency Domain Filtering](968-adaptive-frequency-domain-filtering.md) by its fixed, global ramp filter kernel and explicit geometric cone weighting, rather than adaptive frequency-domain adjustments.

## How it works
The method processes $P$ projections, each of size $W \times H$ pixels. It operates by iterating over each projection, applying geometric cone weighting based on the source-to-detector distance and pixel coordinates, and then applying a 1D ramp filter along the detector rows (horizontal axis) using the Fast Fourier Transform (FFT).

### Inputs and Outputs
**Inputs:**
- `projections`: A flat `std::vector<float>` of size $P \times W \times H$, containing the raw projection data.
- `P`: Integer, number of projections (angles).
- `W`: Integer, detector width in pixels.
- `H`: Integer, detector height in pixels.
- `voxelSize`: Float, physical size of a voxel in the reconstruction volume (mm).
- `sdd`: Float, Source-to-Detector Distance (mm).
- `pixelSize`: Float, physical size of a detector pixel (mm).

**Outputs:**
- `filtered_projections`: A `std::vector<float>` of size $P \times W \times H$, containing the cone-weighted and ramp-filtered projections.

### Data Structures
- `std::vector<float> projections`: Input data layout is row-major: `projections[p * W * H + u * H + v]` corresponds to projection $p$, column $u$, row $v$.
- `std::vector<std::complex<double>>`: Used internally for FFT operations.
- `std::vector<double> ramp_filter`: Pre-computed frequency domain ramp filter kernel.

### Parameters
- `M_PI`: $3.14159265358979323846$.
- `padN`: The smallest power of 2 greater than or equal to $W$ (computed via `nextPow2`).
- Typical values: $P=360, W=512, H=512$ (example dimensions).

### Algorithm Steps

1. **Initialization**:
   - Calculate `projSize` = $W \times H$.
   - Calculate `padN` = `nextPow2(W)`.
   - Build the ramp filter kernel for size `padN` using `buildRampFilter(padN)`.
   - Copy input `projections` to output buffer `out`.

2. **Iterate Projections**:
   - For each projection $p$ from $0$ to $P-1$:
     - Get pointer `proj` to the start of the current projection in `out`.

3. **Cone Weighting**:
   - For each column $u$ from $0$ to $W-1$:
     - Calculate horizontal coordinate $a = -((u - 0.5 \times (W-1)) \times \text{pixelSize})$.
     - For each row $v$ from $0$ to $H-1$:
       - Calculate vertical coordinate $b = ((v - 0.5 \times (H-1)) \times \text{pixelSize})$.
       - Calculate cone weight $cw = \frac{\text{sdd}}{\sqrt{\text{sdd}^2 + a^2 + b^2}}$.
       - Update value: `proj[u * H + v] = (proj[u * H + v] / voxelSize) * cw`.

4. **Ramp Filtering (Row-wise)**:
   - For each row $v$ from $0$ to $H-1$:
     - Extract the row data from `proj` into a temporary vector `row` of size `padN`.
     - Copy `proj[u * H + v]` for $u=0 \dots W-1$ into `row[0 \dots W-1]`.
     - Zero-pad `row[W \dots padN-1]` with zeros.
     - Perform 1D In-place FFT on `row` with sign $-1$ (forward transform).
     - Multiply each element `row[u]` by the corresponding `ramp_filter[u]`.
     - Perform 1D In-place FFT on `row` with sign $+1$ (inverse transform).
     - Calculate normalization factor `norm = 1.0 / (padN * 2.0)`.
     - Update `proj[u * H + v]` with `static_cast<float>(row[u].real() * norm)` for $u=0 \dots W-1$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import cmath

def next_pow2(n):
    """Returns the smallest power of 2 greater than or equal to n."""
    if n <= 0:
        return 1
    p = 1
    while p < n:
        p *= 2
    return p

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
        wlen = cmath.exp(complex(0, sign * 2.0 * math.pi / length))
        for i in range(0, N, length):
            w = complex(1, 0)
            half_len = length // 2
            for j in range(half_len):
                u = a[i + j]
                v = a[i + j + half_len] * w
                a[i + j] = u + v
                a[i + j + half_len] = u - v
                w *= wlen
        length *= 2

def build_ramp_filter(size):
    """
    Creates a ramp filter kernel in the frequency domain.
    Returns a list of real numbers representing the filter.
    """
    f = [0.0] * size
    f[0] = 0.25
    
    # Collect odd indices
    n_odd = []
    for i in range(1, size // 2 + 1, 2):
        n_odd.append(i)
    for i in range(size // 2 - 1, 0, -2):
        n_odd.append(i)
        
    for i, idx in enumerate(n_odd):
        f[1 + i * 2] = -1.0 / (math.pi * idx * math.pi * idx)
        
    # Convert to complex and FFT
    fc = [complex(x, 0) for x in f]
    fft_inplace(fc, -1)
    
    # Multiply by 2 * real part
    for i in range(len(fc)):
        fc[i] = complex(2.0 * fc[i].real, 0)
        
    return [x.real for x in fc]

def apply_fdk_pre_filter(projections, P, W, H, voxel_size, sdd, pixel_size):
    """
    Applies cone weighting and ramp filtering to projections.
    """
    pad_n = next_pow2(W)
    ramp_filter = build_ramp_filter(pad_n)
    
    # Output buffer
    out = list(projections)
    
    for p in range(P):
        # Pointer offset for current projection
        base_idx = p * W * H
        
        # Cone Weighting
        for u in range(W):
            a = -((u - 0.5 * (W - 1)) * pixel_size)
            for v in range(H):
                b = ((v - 0.5 * (H - 1)) * pixel_size)
                cw = sdd / math.sqrt(sdd**2 + a**2 + b**2)
                idx = base_idx + u * H + v
                out[idx] = (out[idx] / voxel_size) * cw
                
        # Ramp Filtering (Row-wise)
        for v in range(H):
            # Extract row and pad
            row = [complex(0, 0)] * pad_n
            for u in range(W):
                idx = base_idx + u * H + v
                row[u] = complex(out[idx], 0)
            # Remaining elements are already 0
            
            # Forward FFT
            fft_inplace(row, -1)
            
            # Apply Ramp Filter
            for u in range(pad_n):
                row[u] = row[u] * complex(ramp_filter[u], 0)
                
            # Inverse FFT
            fft_inplace(row, 1)
            
            # Normalize and write back
            norm = 1.0 / (pad_n * 2.0)
            for u in range(W):
                idx = base_idx + u * H + v
                out[idx] = row[u].real * norm
                
    return out
```

### Edge Cases and Pitfalls
- **Precision**: The implementation uses `double` for FFT intermediate calculations to maintain precision, casting back to `float` for the output.
- **Boundary Conditions**: The ramp filter is applied only to the first $W$ elements after inverse FFT; the zero-padded portion is discarded.
- **Degenerate Input**: If $W$ is already a power of 2, `padN` equals $W$, and no zero-padding occurs.

## Subtopics

### Adaptive Frequency Domain Filtering
Adjusts the filter kernel dynamically based on local frequency content or geometry, unlike the fixed global ramp filter here. Choose this when non-uniform resolution or specific artifact reduction is required. [Adaptive Frequency Domain Filtering](968-adaptive-frequency-domain-filtering.md)

## Operating characteristics
- **Time Complexity**: $O(P \cdot H \cdot W \log W)$ due to the 1D FFT of size $W$ applied to each of the $H$ rows in each of the $P$ projections.
- **Memory Complexity**: $O(P \cdot W \cdot H)$ for storing projections, plus $O(W)$ for temporary row buffers and filter kernel.
- **Performance**: The method is designed for CPU execution, processing all rows in parallel across projections. In the reference implementation, it handles $360 \times 128$ work-items (projections $\times$ rows) efficiently.
- **Strengths**: Simple, deterministic, and easy to implement on CPU. Provides a baseline for GPU acceleration.
- **Limitations**: Fixed filter may not optimally handle all cone angles or noise characteristics compared to adaptive methods. CPU-bound for large datasets.

## Background
This method is part of the CPU-side implementation of the FDK algorithm, specifically mapping the ramp filter kernel strategy used in the Full GPU path. It serves as a reference implementation for validating GPU-based reconstructions. The approach is derived from standard FDK theory, adapted for efficient CPU processing using in-place FFTs.

## Sources
[S1] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
