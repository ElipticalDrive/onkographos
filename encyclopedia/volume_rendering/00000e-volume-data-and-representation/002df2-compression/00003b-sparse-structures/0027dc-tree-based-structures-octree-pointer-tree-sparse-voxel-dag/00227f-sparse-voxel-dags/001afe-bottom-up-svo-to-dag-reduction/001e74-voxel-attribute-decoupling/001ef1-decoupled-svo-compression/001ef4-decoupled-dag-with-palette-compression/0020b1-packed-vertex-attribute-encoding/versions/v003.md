# Packed Vertex Attribute Encoding

Also called: 32-bit vertex packing, Fast Decompression, Packed Vertex Packing with Ambient AO, PackedVertex, Vertex compression, Vertex decompression

## Overview
Packed Vertex Attribute Encoding is a compression scheme for vertex attributes in hardware-accelerated volume splatting that reduces memory usage by packing volume texture coordinates and Gaussian texture coordinates into 32-bit integers per vertex [S1, unofficial]. It sets itself apart from floating-point attribute storage by using discrete bit-fields for metadata, allowing four vertices of a quad to share compact AO data within a single byte [S2, unofficial]. This method is distinct from standard float formats by achieving significant size reduction, e.g., from 40 bytes to 12 bytes per vertex in some implementations [S4, unofficial].

## How it works
**Inputs:**
- Volume texture coordinates ($tx, ty$) for each vertex.
- Gaussian texture coordinate components ($fx, fy$).
- Vertex attributes: normal index, texture index, UV corner index, width, height, ambient occlusion (AO), and color (R, G, B).

**Outputs:**
- A single 32-bit unsigned integer (`uint32_t`) per vertex containing all attributes.

**Data Structures:**
The packed vertex is a `uint32_t`. The bit layout depends on the specific variant (Volume Splatting vs. General Mesh).

*Variant 1: Volume Splatting (S1)*
- Bits 0-7 (8 bits): $ix$ ($tx \mod 256$)
- Bits 8-14 (7 bits): $mx$ ($\lfloor tx / 256 \rfloor$)
- Bits 15-22 (8 bits): $iy$ ($ty \mod 256$)
- Bits 23-29 (7 bits): $my$ ($\lfloor ty / 256 \rfloor$)
- Bit 30 (1 bit): $fx$
- Bit 31 (1 bit): $fy$

*Variant 2: General Mesh with AO/Color (S2)*
- Bits 0-2 (3 bits): `normal_idx`
- Bits 3-10 (8 bits): `tex_index`
- Bits 11-12 (2 bits): `corner_idx`
- Bits 13-16 (4 bits): `width - 1` (saturated)
- Bits 17-20 (4 bits): `height - 1` (saturated)
- Bits 21-22 (2 bits): `ao` (clamped to max 3)
- Bits 23-25 (3 bits): `red` (clamped 0-1, scaled by 7)
- Bits 26-28 (3 bits): `green` (clamped 0-1, scaled by 7)
- Bits 29-31 (3 bits): `blue` (clamped 0-1, scaled by 7)

*Variant 3: Position/Normal/UV (S3)*
- Bits 0-5 (6 bits): x
- Bits 6-11 (6 bits): y
- Bits 12-17 (6 bits): z
- Bits 18-20 (3 bits): `normalIndex`
- Bits 21-22 (2 bits): `uvIndex`
- Bits 23-31: Unused

**Parameters:**
- $mx, my$: 7 bits, allowing volume slices up to $2^{15} \times 2^{15}$ voxels [S1, unofficial].
- $ix, iy$: 8 bits [S1, unofficial].
- $fx, fy$: 1 bit each, binary values (0 or 1) [S1, unofficial].
- Color: 3 bits per channel (R, G, B), 8 discrete levels (0-7) [S2, unofficial].
- AO: 2 bits, 4 discrete levels (0-3) [S2, unofficial].
- Width/Height: 4 bits each, storing values 0-15 corresponding to input widths/heights 1-16 [S2, unofficial].

**Steps:**

1. **Encode Volume Splatting Attributes (S1):**
   For each vertex, decompose volume texture coordinates $tx$ and $ty$:
   $$ ix = tx \mod 256 $$
   $$ mx = \lfloor tx / 256 \rfloor $$
   $$ iy = ty \mod 256 $$
   $$ my = \lfloor ty / 256 \rfloor $$
   Pack into `uint32_t`:
   ```python
   packed = (fy << 31) | (fx << 30) | (my << 23) | (iy << 15) | (mx << 8) | ix
   ```

2. **Encode General Mesh Attributes (S2):**
   - `normal_idx`: Mask with `0x7`.
   - `tex_index`: Mask with `0xFF`.
   - `corner_idx`: Mask with `0x3`.
   - `width_field`: `min(width - 1, 15)`, cast to u32, mask with `0xF`.
   - `height_field`: `min(height - 1, 15)`, cast to u32, mask with `0xF`.
   - `ao_field`: `min(clamp(ao, 0.0, 1.0) * 3, 3)`, cast to u32, mask with `0x3`.
   - `r_field`: `min(clamp(color[0], 0.0, 1.0) * 7, 7)`, cast to u32, mask with `0x7`.
   - `g_field`: `min(clamp(color[1], 0.0, 1.0) * 7, 7)`, cast to u32, mask with `0x7`.
   - `b_field`: `min(clamp(color[2], 0.0, 1.0) * 7, 7)`, cast to u32, mask with `0x7`.
   - Combine:
   ```python
   packed = (b_field << 29) | (g_field << 26) | (r_field << 23) | \
            (ao_field << 21) | (height_field << 17) | (width_field << 13) | \
            (corner_idx << 11) | (tex_index << 3) | normal_idx
   ```

3. **Encode Position/Normal/UV (S3):**
   - Mask x, y, z with `0x3F`.
   - Mask `normalIndex` with `0x7`.
   - Mask `uvIndex` with `0x3`.
   - Combine:
   ```python
   packed = (uvIndex << 21) | (normalIndex << 18) | (z << 12) | (y << 6) | x
   ```

4. **Decompression (S1, S2, S3):**
   - **Volume Splatting:** Extract $mx, my, fx, fy$ using a small lookup table [S1, unofficial]. The lookup table is stored in constant registers and accessed via an address register in the vertex shader [S1, unofficial]. Recover $tx = mx \times 256 + ix$ and $ty = my \times 256 + iy$ [S1, unofficial].
   - **General Mesh:**
   ```python
   normal_idx = packed & 0x7
     tex_index = (packed >> 3) & 0xFF
     corner_idx = (packed >> 11) & 0x3
     width = ((packed >> 13) & 0xF) + 1
     height = ((packed >> 17) & 0xF) + 1
     ao = ((packed >> 21) & 0x3) / 3.0
     r = ((packed >> 23) & 0x7) / 7.0
     g = ((packed >> 26) & 0x7) / 7.0
     b = ((packed >> 29) & 0x7) / 7.0
   ```
   - **Position/Normal/UV:**
   ```python
   x = packed & 0x3F
   y = (packed >> 6) & 0x3F
   z = (packed >> 12) & 0x3F
   normalIndex = (packed >> 18) & 0x7
   uvIndex = (packed >> 21) & 0x3
   ```

**Edge Cases and Pitfalls:**
- **Precision Loss:** Color and AO are quantized. 3-bit color allows only 8 levels; 2-bit AO allows 4 levels [S2, unofficial].
- **Boundary Conditions:** Width and height are stored as `value - 1`. Input width/height must be in [1, 16]. Values outside this range must be saturated/clamped [S2, unofficial].
- **Lookup Table Dependency:** Volume splatting decompression relies on a lookup table for $mx, my, fx, fy$ [S1, unofficial]. The size and content of this table are not explicitly defined in the sources, but it is stored in constant registers [S1, unofficial].

## Subtopics

### [Packed Quad Vertex Format](5062-packed-quad-vertex-format.md)
Packed Quad Vertex Format is a GPU-ready vertex structure designed to minimize memory bandwidth and VRAM usage during procedural mesh generation and hardware-accelerated volume splatting. It achieves this by packing vertex normals, texture coordinates, ambient occlusion (AO) data, and color metadata into a single 32-bit integer field per vertex, alongside a 3-component position vector. This method is distinct from floating-point attribute storage by using discrete bit-fields for metadata, allowing four vertices of a quad to share compact AO data within a single byte.

## Operating characteristics
- **Memory Usage:** Reduces vertex memory significantly. For a $256^3$ volume dataset, vertex memory is reduced from 2048M bytes to 12M bytes (compression ratio 171:1) [S1, unofficial]. In general mesh contexts, vertex format size is reduced from 40 bytes to 12 bytes per vertex [S4, unofficial].
- **Performance:** Decompression and recovery of texture coordinates require only a few operations in programmable vertex shaders [S1, unofficial]. The use of lookup tables allows efficient hardware acceleration [S1, unofficial].
- **Scalability:** Supports volume slices up to $2^{15} \times 2^{15}$ voxels [S1, unofficial].
- **Limitations:** Low precision for color (3-bit/channel) and AO (2-bit) [S2, unofficial]. Limited width/height range (1-16) in the general mesh variant [S2, unofficial].

## Background
Originated in the context of hardware-accelerated adaptive EWA volume splatting [S1, unofficial]. The method was developed to enable interactive rendering of large volume datasets by pre-loading vertex data into local graphics hardware memory [S1, unofficial]. It is used in volume rendering pipelines where memory bandwidth is a bottleneck [S1, unofficial].

## Sources
[S1] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28 (unofficial)
[S2] B4rtekk1/Minerust, https://github.com/B4rtekk1/Minerust, fetched 2026-10-27 (unofficial)
[S3] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S4] DanielWLiu07/voxel-engine, https://github.com/DanielWLiu07/voxel-engine, fetched 2026-10-26 (unofficial)
