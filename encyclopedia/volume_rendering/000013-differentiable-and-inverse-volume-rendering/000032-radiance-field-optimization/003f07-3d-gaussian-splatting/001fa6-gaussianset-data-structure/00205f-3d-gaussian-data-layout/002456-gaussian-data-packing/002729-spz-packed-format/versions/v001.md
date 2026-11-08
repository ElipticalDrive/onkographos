# SPZ Packed Format

Also called: NGSP, Niantic Gaussian Packed

## Overview
SPZ Packed Format is a binary storage format for 3D Gaussian splats designed for low-precision, non-interleaved memory layouts with ZSTD compression support [S1, unofficial]. It solves the problem of efficient serialization and transmission of large-scale Gaussian datasets by separating attributes into parallel buffers, allowing for optimized entropy coding (e.g., g-zipping) and variable spherical harmonics (SH) degrees up to 4 [S1, unofficial]. It is distinguished from interleaved formats by its parallel vector storage and from PNG-based compression by its native binary structure and ZSTD integration [S1, unofficial].

## How it works
**Inputs:** A set of 3D Gaussians defined by position, scale, rotation (quaternion), opacity (alpha), base color, and spherical harmonics coefficients.
**Outputs:** A binary file stream starting with a header, followed by compressed attribute buffers.

**Data Structures:**
The format is identified by the magic number `0x5053474e` (ASCII "SPZN") [S1, unofficial].

1.  **Header (`PackedGaussians`):**
    *   `version` (`uint32_t`): Format version. Default is `LATEST_SPZ_HEADER_VERSION` (4) [S1, unofficial].
    *   `numPoints` (`int32_t`): Total number of Gaussians [S1, unofficial].
    *   `shDegree` (`int32_t`): Degree of spherical harmonics [S1, unofficial].
    *   `fractionalBits` (`int32_t`): Precision bits for fixed-point representations [S1, unofficial].
    *   `antialiased` (`bool`): Flag indicating antialiasing usage [S1, unofficial].
    *   `usesQuaternionSmallestThree` (`bool`): Default `true`. Indicates if SmallestThree quaternion encoding is used [S1, unofficial].
    *   `hadSkippedExtensions` (`bool`): Flag for extension handling [S1, unofficial].
    *   *Note:* In implementations supporting extensions, a conditional field `std::vector<SpzExtensionBasePtr> extensions` may be present, compiled only when `SPZ_BUILD_EXTENSIONS` is defined [S1, unofficial].

2.  **Attribute Buffers:**
    Data is stored in separate `std::vector<uint8_t>` buffers for: `positions`, `scales`, `rotations`, `alphas`, `colors`, and `sh` [S1, unofficial]. These buffers are concatenated or compressed individually.

3.  **Single Gaussian View (`PackedGaussian`):**
    For indexing, a single Gaussian is viewed as a struct of 92 bytes (for SH degree 4):
    *   `position`: `std::array<uint8_t, 9>` (3 components, 3 bytes each) [S1, unofficial].
    *   `rotation`: `std::array<uint8_t, 4>` (4 components, 1 byte each) [S1, unofficial].
    *   `scale`: `std::array<uint8_t, 3>` (3 components, 1 byte each) [S1, unofficial].
    *   `color`: `std::array<uint8_t, 3>` (RGB, 1 byte each) [S1, unofficial].
    *   `alpha`: `uint8_t` [S1, unofficial].
    *   `shR`, `shG`, `shB`: `std::array<uint8_t, 24>` each (24 coefficients for degree 4) [S1, unofficial].

**Decoding Steps:**

1.  **Read Header:** Parse the first 24+ bytes to determine `version`, `numPoints`, `shDegree`, and flags.
2.  **Decompress Buffers:** If `version >= MIN_ZSTD_SPZ_HEADER_VERSION` (4), decompress the attribute buffers using ZSTD [S1, unofficial].
3.  **Decode Attributes:** Iterate through each Gaussian index $i$ from $0$ to `numPoints` - 1.

    **3a. Position, Scale, Color, Alpha:**
    These are typically stored as fixed-point or quantized uint8 values. The specific quantization depends on `fractionalBits` and implementation details not fully specified in the core notes, but generally involve scaling uint8 values to float ranges [S1, unofficial].

    **3b. Rotation (Quaternion):**
    The encoding method depends on `usesQuaternionSmallestThree` and `version`.

    *   **SmallestThree (Default for Version 3+):**
        1.  Pack 4 bytes into `uint32_t comp`.
        2.  Extract `i_largest = comp >> 30`.
        3.  Initialize `sum_squares = 0.0f`.
        4.  Loop `i` from 3 down to 0:
            *   If `i != i_largest`:
                *   Extract `mag = comp & 0x1FF`.
                *   Extract `negbit = (comp >> 9) & 1`.
                *   Shift `comp` right by 10 bits.
                *   Calculate `rotation[i] = sqrt(0.5f) * mag / 511.0f`.
                *   If `negbit` is 1, `rotation[i] = -rotation[i]`.
                *   `sum_squares += rotation[i] * rotation[i]`.
        5.  Calculate `rotation[i_largest] = sqrt(max(0.0f, 1.0f - sum_squares))`. [G: Guard against numerical error ensuring non-negative argument for sqrt].
        6.  Apply `c.flipQ` to the first 3 components of the rotation vector.
        7.  If `c.rotFlipQFunc` exists, call it [S1, unofficial].

    *   **FirstThree (Legacy):**
        1.  Convert the first 3 uint8s to floats.
        2.  Scale by `1/127.5`.
        3.  Subtract 1.
        4.  Multiply by `c.flipQ`.
        5.  Copy to `rotation[0..2]`.
        6.  Calculate `rotation[3] = sqrt(max(0.0f, 1.0f - (rotation[0]^2 + rotation[1]^2 + rotation[2]^2)))`. [G: Guard against numerical error].
        7.  If `c.rotFlipQFunc` exists, call it [S1, unofficial].

    **3c. Spherical Harmonics (SH):**
    1.  Read `SH_MAX_COEFFS` (24 for degree 4) uint8 values for each R, G, B channel.
    2.  Unquantize each coefficient $x$ using:
        $$v = \frac{x - 128.0}{128.0}$$
        where $x$ is the uint8 value [S1, unofficial].

    **3d. Alpha and Color:**
    1.  Alpha is typically a uint8 representing opacity.
    2.  Color is unquantized from uint8 to [0, 1] range.
    3.  Apply base color scale factor: $C_{base} = C_{raw} \times 0.15$. This allows out-of-range base colors that higher SH bands can correct [S1, unofficial].

**Parameters:**
*   `SH_MAX_COEFFS`: 24 (for degree 4) [S1, unofficial].
*   `DEFAULT_SH1_BITS`: 5 (bits for first SH coefficient) [S1, unofficial].
*   `DEFAULT_SH_REST_BITS`: 4 (bits for remaining SH coefficients) [S1, unofficial].
*   `LATEST_SPZ_HEADER_VERSION`: 4 [S1, unofficial].
*   `MIN_ZSTD_SPZ_HEADER_VERSION`: 4 [S1, unofficial].
*   `MIN_SMALLEST_THREE_QUATERNIONS_VERSION`: 3 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Quaternion Reconstruction:** If numerical error causes $1 - \sum_{i=0}^{2} r_i^2 < 0$, `std::max(0.0f, ...)` ensures the sqrt argument is non-negative [S1, unofficial].
*   **WASM Memory View:** In JavaScript/WebAssembly implementations, the callback must consume the chunk immediately because the typed memory view detaches if any subsequent WASM allocation grows the heap [S1, unofficial].
*   **Version Compatibility:** Decoders must check `version` to determine if ZSTD is used (>=4) and which quaternion encoding is active (>=3 uses SmallestThree) [S1, unofficial].

## Subtopics

### SPZ Packed Data Layout
A variant configuration focusing on the packing options for SH coefficients. It uses `PackOptions::sh1Bits` (defaulting to `DEFAULT_SH1_BITS`=5) and `PackOptions::shRestBits` (defaulting to `DEFAULT_SH_REST_BITS`=4) to specify quantization bit depths for SH degree 1 and degree 2+ coefficients respectively. This reduces entropy for g-zipping during the packing phase [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Decompression is linear in the number of bytes, dominated by ZSTD decompression and per-Gaussian decoding. Quaternion decoding is constant time per Gaussian.
*   **Memory Complexity:** Linear in the number of Gaussians. The non-interleaved layout allows for cache-friendly access patterns when processing specific attributes (e.g., all positions) but requires managing multiple buffers.
*   **Performance:** ZSTD compression provides significant size reduction, especially for correlated Gaussian data. SmallestThree quaternion encoding offers better precision for small components compared to FirstThree, at the cost of slightly more complex decoding logic [S1, unofficial].
*   **Strengths:** Low precision storage, efficient compression, support for high-degree SH, flexible extension system.
*   **Limitations:** Requires custom decoder; non-interleaved layout may be less convenient for some rendering pipelines that expect interleaved data; WASM memory management requires careful handling [S1, unofficial].

## Background
SPZ Packed Format was developed by Niantic Labs as part of their open-source SPZ library for efficient storage and transmission of 3D Gaussian Splatting data [S1, unofficial]. It evolved to support higher precision requirements (SmallestThree quaternions) and compression (ZSTD) in later versions. The format is used in Niantic's AR/VR applications and open-source 3D Gaussian Splatting toolchains [S1, unofficial].

## Sources
[S1] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
