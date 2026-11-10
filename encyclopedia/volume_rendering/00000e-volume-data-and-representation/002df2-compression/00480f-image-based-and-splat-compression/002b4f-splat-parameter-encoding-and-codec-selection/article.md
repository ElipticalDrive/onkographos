# Splat parameter encoding and codec selection

Also called: Luxar store compression, quantised splat encoding

## Overview
This method defines the binary layout and compression pipeline for 3D Gaussian Splat parameters to minimize storage size while maintaining rendering fidelity. It solves the problem of excessive file sizes in raw Gaussian Splat formats (typically 30–37 bytes/splat) by applying targeted quantization and a width-aware codec policy. It differs from generic image compression [PngCompression](11785-pngcompression.md) by operating on structured, heterogeneous vector fields (position, scale, rotation, amplitude) rather than uniform pixel grids, and by using spatial bucketing and delta encoding specific to splat distributions.

## How it works
**Inputs:** Float32 Gaussian Splat parameters (centres, Cholesky factors for rotation/scale, amplitudes).
**Outputs:** Compressed `.ksplat` byte stream.

**Data Structures and Layout:**
The store consists of four parameter arrays, encoded as follows [S1, unofficial]:
1.  **Centres:** `uint16` integers.
2.  **Cholesky Factors:** `uint8` integers, certified for validity.
3.  **Amplitudes:** Selected based on the array's dynamic range:
    *   Bounded-linear `uint8` or `uint16`.
    *   Geometric-log `uint16` grid.

**Algorithm Steps:**
1.  **Quantization:** Convert float32 parameters to the target integer types. The quantization fidelity is bounded: PSNR loss vs. original is at most $-0.679$ dB (median $-0.012$ dB) [S1, unofficial].
2.  **Preprocessing (Centres):** Apply columnar delta prefiltering to the centre coordinates to improve compressibility [S1, unofficial].
3.  **Chunking:** Divide each parameter array into chunks aligned to atom boundaries (to mitigate specific codec anomalies) [S1, unofficial].
4.  **Codec Selection:** Apply the "shipped rule" policy using **Blosc-zstd level 9** [S1, unofficial]:
    *   **Byte Shuffle:** Apply to multi-byte integer codes.
    *   **No Byte Shuffle:** Skip for single-byte codes (`uint8`) and floating-point arrays.
    *   **Amplitude Optimization:** In the optimized configuration, leave log-companded amplitude codes unshuffled to reduce size by 1.44% [S1, unofficial].
5.  **Compression:** Compress each chunk using the selected Blosc configuration.

**Parameters:**
*   **Codec:** Blosc-zstd level 9 [S1, unofficial].
*   **Chunk Size:** Atom-aligned (default mitigates 64 KiB shuffle anomaly); standard 64 KiB chunks used for baseline comparison [S1, unofficial].

**Pitfalls:**
*   **Shuffle Anomaly:** Level-dependent Blosc shuffle anomalies occur on 64 KiB chunks. Mitigation requires atom-aligned chunks and columnar delta prefiltering on centres [S1, unofficial].
*   **Precision:** Quantised render PSNR vs. float32 render ranges from 59–88 dB [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_splats(centres_f32, cholesky_f32, amplitudes_f32):
    # 1. Quantization
    centres_u16 = quantize_uint16(centres_f32)
    cholesky_u8 = quantize_uint8(cholesky_f32)
    
    # Determine amplitude encoding based on dynamic range
    if dynamic_range(amplitudes_f32) < threshold_linear:
        amps_u8 = quantize_bounded_linear_uint8(amplitudes_f32)
    else:
        amps_u16 = quantize_geometric_log_uint16(amplitudes_f32)

    # 2. Preprocessing
    # Columnar delta prefiltering on centres
    centres_delta = columnar_delta_encode(centres_u16)

    # 3. Chunking and Compression Configuration
    # Atom-aligned chunks to avoid 64 KiB shuffle anomaly
    chunk_size = get_atom_aligned_chunk_size() 
    
    # Codec policy: Blosc-zstd level 9
    config = {
        'codec': 'zstd',
        'level': 9,
        'shuffle': 'multi_byte_int' # Apply to multi-byte, skip single-byte/float
    }

    # 4. Compress Arrays
    # Centres: Delta encoded, multi-byte (uint16), so shuffle ON
    centres_compressed = blosc_compress(centres_delta, config)
    
    # Cholesky: Single-byte (uint8), so shuffle OFF
    config['shuffle'] = 'none'
    cholesky_compressed = blosc_compress(cholesky_u8, config)
    
    # Amplitudes: Optimized config leaves log-companded codes unshuffled
    # If using geometric-log uint16, it is multi-byte, but optimized config skips shuffle
    if use_optimized_config:
        config['shuffle'] = 'none'
    
    if is_geometric_log(amps_u16):
        amps_compressed = blosc_compress(amps_u16, config)
    else:
        # Bounded linear uint8, already no shuffle
        amps_compressed = blosc_compress(amps_u8, config)

    return pack_store(centres_compressed, cholesky_compressed, amps_compressed)
```

## Subtopics

### [Ksplat Decompression](6899-ksplat-decompression.md)
Ksplat Decompression reads and decompresses `.ksplat` files, which contain spatially bucketed and quantized 3D Gaussian Splat data, into a structured `DataTable` format suitable for rendering. It handles position quantization, scale/rotation compression, and spherical harmonics decoding. Unlike raw PLY formats, it supports three distinct compression modes (Float32, Float16, Uint8) to balance file size and precision, and uses spatial bucketing to optimize position storage.

## Operating characteristics
**Time and Memory Complexity:**
*   **Store Size:** 8.8–12.7 bytes/splat (down from 30–37 bytes/splat) [S1, unofficial].
*   **Shipped Rule Storage:** 9.63 bytes/splat. If re-chunked to 64 KiB without prefilter, 10.09 bytes/splat (14.3% below uniform Blosc-zstd level-3 bit-shuffle baseline) [S1, unofficial].
*   **Optimized Unshuffled Config:** 9.50 bytes/splat (15.6% below baseline at 64 KiB) [S1, unofficial].

**Measured Performance:**
*   **Hardware:** Benchmark workstation (identical chunk sets) [S1, unofficial].
*   **Decoding Throughput:**
    *   In-browser (WebAssembly): 138–215 MB/s [S1, unofficial].
    *   Native: 326–592 MB/s [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant size reduction with minimal PSNR loss (median $-0.012$ dB). Fast loading times when converted to `.ksplat` due to cache-optimized ordering [S2, unofficial].
*   **Limitations:** Requires specific chunk alignment to avoid codec anomalies. Progressive loading of `.ply`/`.splat` skips these optimizations [S2, unofficial].

## Background
This method originates from the Luxar project's work on scalable interactive web visualization of multidimensional scientific data, specifically focusing on Gaussian Splatting for microscopy [S1, unofficial]. The `.ksplat` format was introduced by mkkellogg in the GaussianSplats3D library as a trimmed-down, compressed version of original `.ply` files to match internal splat data formats for fastest loading [S2, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
