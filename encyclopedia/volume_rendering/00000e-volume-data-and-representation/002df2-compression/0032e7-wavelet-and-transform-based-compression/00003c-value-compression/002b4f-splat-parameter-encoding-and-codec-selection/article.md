# Splat parameter encoding and codec selection

Also called: Luxar store compression, quantised splat encoding

## Overview
This method defines the storage format and compression pipeline for 3D Gaussian Splat parameters (centres, scales, rotations, and amplitudes) to minimize file size while maintaining rendering fidelity. It solves the problem of efficiently storing and streaming large-scale Gaussian splat scenes by applying specific quantization schemes to each parameter type and selecting an optimal codec configuration. It differs from raw PLY formats by using spatial bucketing and specialized quantization to achieve significantly smaller file sizes (8.8–12.7 bytes/splat vs 30–37 bytes/splat) [S1, unofficial]. It is distinct from progressive loading of standard `.ply` or `.splat` files, which lack cache-optimized ordering and specific compression policies [S2, unofficial].

## How it works
**Inputs:** A set of 3D Gaussian splats defined by float32 parameters: position ($\mathbf{p}$), scale ($\mathbf{s}$), rotation ($\mathbf{r}$), and spherical harmonics amplitudes ($\mathbf{a}$).
**Outputs:** A compressed binary store (`.ksplat`) containing quantized and encoded parameter arrays.

**Data Structures and Layout:**
The store organizes splat parameters into four arrays, processed with width-aware per-dtype compression policies [S1, unofficial].
1.  **Centres:** Stored as `uint16`. These represent the spatial positions.
2.  **Cholesky Factors:** Stored as `uint8`. These encode the rotation and scale (covariance matrix) via Cholesky decomposition of the 3x3 covariance matrix. The values are "certified" to fit within the `uint8` range [S1, unofficial].
3.  **Amplitudes:** Stored as either bounded-linear `uint8`/`uint16` or geometric-log `uint16`. The selection depends on the array's dynamic range [S1, unofficial].
4.  **Spherical Harmonics (SH):** Not explicitly detailed in the provided notes for the core encoding layout, but implied to be part of the parameter set handled by the codec sweep [S1, unofficial].

**Steps:**
1.  **Quantization:**
    *   Centres are quantized to `uint16`.
    *   Cholesky factors are quantized to `uint8`.
    *   Amplitudes are quantized to `uint8` or `uint16` based on dynamic range analysis.
    *   *Note:* The quantization fidelity results in a PSNR loss of at most $-0.679$ dB (median $-0.012$ dB) against the original float32 fits [S1, unofficial].

2.  **Preprocessing (Prefiltering):**
    *   Apply columnar delta prefiltering to the Centres array. This step is enabled by a probe to optimize compression efficiency [S1, unofficial].

3.  **Chunking:**
    *   Divide the data into chunks. The shipped configuration uses atom-aligned chunks to mitigate specific codec anomalies [S1, unofficial].
    *   *Pitfall:* A level-dependent Blosc shuffle anomaly occurs on 64 KiB chunks. This is mitigated by using atom-aligned chunks and the aforementioned columnar delta prefiltering on centres [S1, unofficial].

4.  **Codec Selection and Encoding:**
    *   Use **Blosc** with **zstd** compressor at **level 9** [S1, unofficial].
    *   Apply **Byte Shuffle** for multi-byte integer codes (e.g., `uint16` centres).
    *   **Skip Byte Shuffle** for single-byte codes (e.g., `uint8` Cholesky factors) and floating-point arrays [S1, unofficial].
    *   *Optimization:* In the optimized configuration, leave log-companded amplitude codes unshuffled to reduce size by 1.44% [S1, unofficial].

5.  **Storage:**
    *   Write the encoded chunks to the `.ksplat` file.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_splats(splats):
    # splats: list of dicts with keys 'pos' (vec3), 'cov' (3x3 matrix), 'amp' (float)
    
    # 1. Quantize Centres to uint16
    centres_uint16 = quantize_uint16([s['pos'] for s in splats])
    
    # 2. Quantize Cholesky Factors to uint8
    # Cholesky decomposition of covariance matrix, then quantize to uint8
    cholesky_uint8 = quantize_cholesky_uint8([s['cov'] for s in splats])
    
    # 3. Quantize Amplitudes
    # Select bounded-linear uint8/uint16 or geometric-log uint16 based on dynamic range
    amps_dtype, amps_quantized = select_and_quantize_amplitudes([s['amp'] for s in splats])
    
    # 4. Preprocess Centres
    # Apply columnar delta prefiltering to mitigate 64 KiB chunk anomalies
    centres_prefiltered = columnar_delta_prefilter(centres_uint16)
    
    # 5. Encode with Blosc-zstd level 9
    # Width-aware per-dtype compressor policy
    encoded_centres = blosc_compress(
        data=centres_prefiltered,
        compressor='zstd',
        level=9,
        shuffle=True,  # Multi-byte integer
        typesize=2
    )
    
    encoded_cholesky = blosc_compress(
        data=cholesky_uint8,
        compressor='zstd',
        level=9,
        shuffle=False, # Single-byte integer
        typesize=1
    )
    
    # Amplitude encoding depends on dtype selection
    if amps_dtype == 'uint8':
        encoded_amps = blosc_compress(
            data=amps_quantized,
            compressor='zstd',
            level=9,
            shuffle=False, # Single-byte integer
            typesize=1
        )
    elif amps_dtype == 'uint16':
        # Optimized config: leave log-companded amplitude codes unshuffled
        encoded_amps = blosc_compress(
            data=amps_quantized,
            compressor='zstd',
            level=9,
            shuffle=False, # Unshuffled for log-companded
            typesize=2
        )
    else:
        raise ValueError("Unsupported amplitude dtype")
        
    # 6. Write to .ksplat file
    write_ksplat_file(encoded_centres, encoded_cholesky, encoded_amps)
```

**Parameters:**
*   **Codec:** Blosc-zstd level 9 [S1, unofficial].
*   **Chunk Size:** Atom-aligned (specific size not given, but anomaly noted at 64 KiB) [S1, unofficial].
*   **Shuffle Policy:** True for multi-byte integers, False for single-byte integers and floats [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **64 KiB Chunk Anomaly:** Blosc shuffle performance degrades at 64 KiB chunks. Mitigation: Use atom-aligned chunks and columnar delta prefiltering on centres [S1, unofficial].
*   **Dynamic Range:** Incorrect selection of amplitude quantization (linear vs. geometric-log) can lead to suboptimal compression or precision loss. The selection is based on the array's dynamic range [S1, unofficial].

## Subtopics

### [Ksplat Decompression](6899-ksplat-decompression.md)
Ksplat Decompression reads and decompresses `.ksplat` files, which contain spatially bucketed and quantized 3D Gaussian Splat data, into a structured `DataTable` format suitable for rendering. It handles position quantization, scale/rotation compression, and spherical harmonics decoding. Unlike raw PLY formats, it supports three distinct compression modes (Float32, Float16, Uint8) to balance file size and precision, and uses spatial bucketing to optimize position storage.

## Operating characteristics
**Time and Memory Complexity:**
*   **Compression:** Not given in the sources.
*   **Decompression:** Decoding throughput is 138–215 MB/s in-browser (via WebAssembly) and 326–592 MB/s natively on a benchmark workstation, on identical chunk sets [S1, unofficial].

**Measured Performance:**
*   **Store Size:** 8.8–12.7 bytes/splat, down from 30–37 bytes/splat for float32 [S1, unofficial].
*   **Shipped Rule Storage:** 9.63 bytes/splat. If re-chunked to 64 KiB without prefilter, 10.09 bytes/splat (14.3% below uniform Blosc-zstd level-3 bit-shuffle baseline) [S1, unofficial].
*   **Optimized Unshuffled Config:** 9.50 bytes/splat (15.6% below baseline at 64 KiB) [S1, unofficial].
*   **Quantisation PSNR Loss:** At most $-0.679$ dB (median $-0.012$ dB) vs original float32 fits [S1, unofficial].
*   **Quantised Render PSNR:** 59–88 dB vs float32 render [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in file size (up to ~60% reduction) while maintaining high rendering PSNR. Optimized for fast loading times by matching internal splat data format [S2, unofficial].
*   **Limitations:** Requires specific codec configuration (Blosc-zstd level 9) and prefiltering to avoid performance anomalies. In-browser decoding is slower than native decoding [S1, unofficial].

**When to Use:**
*   Use for storing and streaming large-scale 3D Gaussian Splat scenes where file size and loading speed are critical.
*   Prefer over `.ply` or `.splat` files for optimal performance, as they lack cache-optimized ordering and specific compression [S2, unofficial].

## Background
This method originates from the Luxar project, which focuses on Gaussian splatting for microscopy and scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. The `.ksplat` format was introduced as a trimmed-down and compressed version of original `.ply` files, designed for fastest loading times by matching the internal format for splat data [S2, unofficial]. The encoding and codec selection details were documented in supplementary documents accompanying the Luxar research paper [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
