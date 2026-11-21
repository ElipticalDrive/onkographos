# Gaussian Splat Data Packing and Compression

Also called: Gaussian Splat Encoding, Runtime Splat Processing, SPZ Processing

## Overview
This method converts raw Gaussian splat data (typically from SPZ or PLY files) into a compressed, GPU-ready byte buffer layout optimized for real-time rendering. It solves the problem of high memory bandwidth and storage costs associated with storing full-precision floating-point attributes for millions of splats. It sets itself apart from sibling methods by integrating Morton space-filling curve reordering for cache coherence with aggressive per-attribute quantization (Norm11, Norm16, Norm8) and chunk-based storage for spherical harmonics, all implemented in a Burst-compiled, runtime-safe manner [S1, unofficial].

## How it works
**Inputs:**
- Raw splat data array (`NativeArray<InputSplatData>`), containing position, scale, rotation, color, and spherical harmonic (SH) coefficients.
- Format specifications for each attribute: `posFormat`, `scaleFormat`, `colorFormat`, `shFormat`.

**Outputs:**
- Compressed GPU-ready byte buffers stored in `RuntimeSplatData`.
- Reordered splat indices based on Morton codes.

**Data Structures:**
1.  **InputSplatData**: The raw input structure. Fields are not explicitly detailed in the source, but implied to contain `float3` position, `float3` scale, `float4` color, and SH coefficients.
2.  **GaussianSplatAsset.ChunkInfo**: Used when SH data is not stored in full float32. Fields are not explicitly detailed, but used to define chunk boundaries for SH data.
3.  **ReorderMortonJob**: A job struct for parallel Morton reordering.
    -   `const float kScaler = (1<<21)-1`
    -   `float3 m_BoundsMin`
    -   `float3 m_InvBoundsSize`
    -   `ReadOnly NativeArray<InputSplatData> m_SplatData`
    -   `NativeArray<(ulong, int)> m_Order` (stores Morton code and original index)
4.  **CalcChunkDataJob**: A job struct for computing SH chunks.
    -   `NativeArray<InputSplatData> splatData`
    -   `NativeArray<GaussianSplatAsset.ChunkInfo> chunks`

**Parameters (Defaults):**
-   `posFormat`: `GaussianSplatAsset.VectorFormat.Norm11` [S1, unofficial]
-   `scaleFormat`: `GaussianSplatAsset.VectorFormat.Norm11` [S1, unofficial]
-   `colorFormat`: `GaussianSplatAsset.ColorFormat.Norm8x4` [S1, unofficial]
-   `shFormat`: `GaussianSplatAsset.SHFormat.Norm6` [S1, unofficial]

**Steps:**

1.  **Input Validation**: Check if the input splat array is empty. If empty, throw `ArgumentException` [S1, unofficial].
2.  **Scene Bounds Calculation**: Iterate through all splats to find the axis-aligned bounding box (AABB).
    -   Initialize `bMin` to positive infinity and `bMax` to negative infinity.
    -   For each splat position `p`:
        -   `bMin = min(bMin, p)`
        -   `bMax = max(bMax, p)`
3.  **Morton Reordering**:
    -   Calculate inverse bounds size: `invSize = 1.0f / (bMax - bMin)`.
    -   For each splat, normalize position to `[0, 1]`:
        $$ pos_{norm} = (p - bMin) \times invSize $$
    -   Scale to integer range for Morton encoding:
        $$ ipos = \text{cast\_to\_uint3}(pos_{norm} \times kScaler) $$
        where $kScaler = (1 \ll 21) - 1$ [S1, unofficial].
    -   Generate Morton code: $code = \text{MortonEncode3}(ipos)$ [S1, unofficial].
    -   Sort splats by `code`.
    -   Reorder the `InputSplatData` array according to the sorted order.
4.  **Chunk Data Creation (Conditional)**:
    -   If any format is not full float32 (`posFormat != Float32` OR `scaleFormat != Float32` OR `colorFormat != Float32x4` OR `shFormat != Float32`), call `CreateChunkData` [S1, unofficial].
    -   This step likely groups splats into chunks for efficient SH storage, ensuring chunk bounds are clamped to `splatData.Length` to avoid out-of-bounds access [S1, unofficial].
5.  **Position Encoding**: Call `CreatePositionsData` with the reordered splats and `posFormat`.
    -   If `posFormat` is `Norm11`:
        $$ enc = (v.x \times 2047.5f) \mid (v.y \times 1023.5f \ll 11) \mid (v.z \times 2047.5f \ll 21) $$
        Stored as 4 bytes (uint) [S1, unofficial].
    -   If `posFormat` is `Norm16`:
        $$ enc = (v.x \times 65535.5f) \mid (v.y \times 65535.5f \ll 16) \mid (v.z \times 65535.5f \ll 32) $$
        Stored as 4 bytes (uint) + 2 bytes (ushort) [S1, unofficial].
6.  **Scale and SH Index Encoding**: Call `CreateOtherData` with splats, `scaleFormat`, and `splatSHIndices`.
    -   Scale encoding follows similar quantization rules as position based on `scaleFormat` (default `Norm11`).
7.  **Color Encoding**: Call `CreateColorData` with splats.
    -   Default format `Norm8x4` implies 8-bit per channel quantization for RGBA.
8.  **SH Data Encoding**: Call `CreateSHData` with splats, `shFormat`, and `clusteredSHs`.
    -   Default format `Norm6` implies 6-bit quantization for SH coefficients.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_gaussian_splats(splat_data: List[Splat], pos_format: str = "Norm11", scale_format: str = "Norm11", color_format: str = "Norm8x4", sh_format: str = "Norm6") -> RuntimeSplatData:
    if not splat_data:
        raise ArgumentException("Input splat array is empty")

    # Step 2: Calculate bounds
    b_min = float3(inf, inf, inf)
    b_max = float3(-inf, -inf, -inf)
    for splat in splat_data:
        b_min = min(b_min, splat.position)
        b_max = max(b_max, splat.position)

    inv_bounds_size = 1.0 / (b_max - b_min)
    k_scaler = (1 << 21) - 1

    # Step 3: Morton Reordering
    indexed_splats = []
    for i, splat in enumerate(splat_data):
        pos_norm = (splat.position - b_min) * inv_bounds_size
        ipos = uint3(pos_norm * k_scaler)
        morton_code = morton_encode_3(ipos)
        indexed_splats.append((morton_code, i))

    indexed_splats.sort(key=lambda x: x[0])
    reordered_splat_data = [splat_data[idx] for _, idx in indexed_splats]

    # Step 4: Chunk Data (if needed)
    needs_chunks = (pos_format != "Float32" or scale_format != "Float32" or 
                    color_format != "Float32x4" or sh_format != "Float32")
    chunks = []
    if needs_chunks:
        chunks = create_chunk_data(reordered_splat_data)
        # Ensure chunk bounds are clamped to splat_data.Length
        for chunk in chunks:
            chunk.end = min(chunk.end, len(reordered_splat_data))

    # Step 5: Encode Positions
    pos_buffer = bytearray()
    for splat in reordered_splat_data:
        if pos_format == "Norm11":
            enc = (splat.position.x * 2047.5) | \
                  (splat.position.y * 1023.5 << 11) | \
                  (splat.position.z * 2047.5 << 21)
            pos_buffer += enc.to_bytes(4, byteorder='little')
        elif pos_format == "Norm16":
            enc = (splat.position.x * 65535.5) | \
                  (splat.position.y * 65535.5 << 16) | \
                  (splat.position.z * 65535.5 << 32)
            # Stored as 4 bytes uint + 2 bytes ushort
            pos_buffer += (enc & 0xFFFFFFFF).to_bytes(4, byteorder='little')
            pos_buffer += ((enc >> 32) & 0xFFFF).to_bytes(2, byteorder='little')
        else:
            # Handle other formats or default to Float32 if not specified
            pass

    # Step 6: Encode Scale and SH Indices
    scale_buffer = bytearray()
    sh_indices_buffer = bytearray()
    for splat in reordered_splat_data:
        # Scale encoding similar to position based on scale_format
        if scale_format == "Norm11":
            enc = (splat.scale.x * 2047.5) | \
                  (splat.scale.y * 1023.5 << 11) | \
                  (splat.scale.z * 2047.5 << 21)
            scale_buffer += enc.to_bytes(4, byteorder='little')
        # SH indices are stored separately
        sh_indices_buffer += splat.sh_index.to_bytes(4, byteorder='little')

    # Step 7: Encode Color
    color_buffer = bytearray()
    for splat in reordered_splat_data:
        if color_format == "Norm8x4":
            # 8-bit per channel RGBA
            r = int(splat.color.r * 255)
            g = int(splat.color.g * 255)
            b = int(splat.color.b * 255)
            a = int(splat.color.a * 255)
            color_buffer += bytes([r, g, b, a])

    # Step 8: Encode SH Data
    sh_buffer = bytearray()
    for splat in reordered_splat_data:
        if sh_format == "Norm6":
            # 6-bit quantization for SH coefficients
            # Implementation details for Norm6 quantization are not fully specified in sources
            # Assuming standard quantization: coeff * (2^5 - 1)
            for coeff in splat.sh_coefficients:
                q = int(coeff * 31.5)
                sh_buffer += q.to_bytes(1, byteorder='little') # Simplified, actual packing may vary

    return RuntimeSplatData(pos_buffer, scale_buffer, sh_indices_buffer, color_buffer, sh_buffer, chunks)
```

## Operating characteristics
-   **Time Complexity**: Dominated by the Morton sort, which is $O(N \log N)$ where $N$ is the number of splats. Bounds calculation is $O(N)$. Encoding is $O(N)$.
-   **Memory Complexity**: Significantly reduced compared to raw float32 storage. Position and scale drop from 12 bytes each to 4 bytes (Norm11) or 6 bytes (Norm16). Color drops from 16 bytes to 4 bytes (Norm8x4). SH data is heavily compressed via quantization and chunking.
-   **Performance**: The method is designed for runtime execution using Burst compilation, ensuring zero Editor dependencies and safe usage in builds [S1, unofficial].
-   **Strengths**: High compression ratio, cache-friendly ordering via Morton codes, runtime-safe.
-   **Limitations**: Quantization introduces precision loss. Morton reordering requires sorting, which can be costly for very large datasets if not parallelized effectively. Empty input arrays cause exceptions [S1, unofficial].

## Background
This method is part of the WorldLabs Unity integration package, specifically designed for processing SPZ (Splat Zone) files generated by WorldLabs' AI world generation API. It enables efficient runtime rendering of Gaussian Splat scenes in Unity, particularly for VR and mobile platforms [S1, unofficial]. The approach reflects the broader trend in 3D Gaussian Splatting towards optimizing data layouts for real-time rendering engines.

## Sources
[S1] nigelhartman/worldlabs_unity, https://github.com/nigelhartman/worldlabs_unity, fetched 2026-10-27 (unofficial)
