# Voxel data loading and quantization

Also called: 16-bit voxel quantization, Stanford model loader, Volume Data Loading and Storage

## Overview
This method defines the procedure for ingesting raw binary volume data (typically from CT scanners or specific model formats) into memory, applying domain-specific quantization to reduce precision, and preparing the data for GPU transfer or further processing. It solves the problem of converting high-precision raw sensor data (e.g., 12-bit Hounsfield units) into a format suitable for real-time raycasting or sparse voxel octree construction, often involving thresholding and bit-depth reduction. It differs from generic loaders by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline rather than treating data as opaque bytes.

## How it works
The method consists of two primary variants found in the literature: a specific implementation for the Stanford Bunny/CT data (S1, S2) and a general VolPack interface (S3), plus a specific 16-to-8-bit conversion for VDB data (S4).

### Inputs and Outputs
*   **Input:** Raw binary files (e.g., `.zaz` slices) or a memory pointer to a voxel array. Data types are typically 16-bit integers (`uint16_t` or `short`).
*   **Output:** A 3D array or GPU buffer containing quantized values (8-bit or reduced 16-bit), ready for raycasting or sparse storage.

### Data Structures
1.  **Raw Slice Array:** A 2D array of `uint16_t` representing a single slice (e.g., 512x512) [S1, unofficial].
2.  **Volume Array:** A 3D array or linear buffer. For the Stanford Bunny, dimensions are $512 \times 512 \times 360$ [S2, unofficial].
3.  **VolPack Voxel Structure:** Defined by strides. The caller allocates memory; VolPack only stores pointers and stride values [S3].
4.  **VDB Header/Data:**
    *   Header: 16-bit resolution at offsets 0, 2, 4; 32-bit node count at offset 12 [S4, unofficial].
    *   Coordinates: Int32 array starting at offset 32 [S4, unofficial].
    *   Node Data: Uint16 array following coordinates [S4, unofficial].

### Steps and Formulas

#### Variant A: Stanford Bunny / CT Scan Loading (S1, S2)
1.  **Read Slices:** Load each binary slice file into a 2D `uint16_t` array. The file extension is typically `.zaz` [S1, unofficial].
2.  **Assemble Volume:** Stack slices along the Z-axis. The value at position $(x, y, z)$ is accessed as `data[y][z * depth + x]` [S1, unofficial].
3.  **Quantize and Threshold:** Apply the following transformation to each voxel value $v_{in}$:
    $$
    v_{out} = \begin{cases} 
    0 & \text{if } v_{in} < 160 \\
    \lfloor \frac{v_{in}}{4096} \rfloor \cdot 16 & \text{otherwise}
    \end{cases}
    $$
    *   **Threshold:** Values below 160 (approx. 0xa0) are set to 0 [S1, unofficial].
    *   **Quantization:** The value is divided by 4096 (equivalent to right-shifting by 12 bits, retaining the top 4 bits of the 12-bit data), then multiplied by 16 (left-shifting by 4 bits) to align with 8-bit or 16-bit integer ranges [S1, unofficial]. Note: S1 describes this as `value /= 256 * 16; value *= 16`, which mathematically simplifies to dividing by 256 then multiplying by 16, but the formula provided in S1 explicitly uses 4096 ($256 \times 16$) as the divisor. The formula $\lfloor v_{in} / 4096 \rfloor \cdot 16$ preserves the most significant 4 bits of the 12-bit Hounsfield data [S1, unofficial].
4.  **Transfer:** Copy the resulting 3D array to GPU memory via OpenCL or similar API [S2, unofficial].

#### Variant B: VolPack Raw Voxel Setup (S3)
1.  **Allocate Memory:** Allocate a contiguous block of memory for the voxel data.
2.  **Define Strides:** Calculate byte strides for X, Y, and Z dimensions.
    *   $xstride$: Bytes between adjacent voxels along X.
    *   $ystride$: Bytes between adjacent voxels along Y.
    *   $zstride$: Bytes between adjacent voxels along Z.
3.  **Register Data:** Call `vpSetRawVoxels(vpc, voxels, size, xstride, ystride, zstride)`.
    *   `voxels`: Pointer to the data.
    *   `size`: Total size in bytes.
    *   Strides determine access patterns. Z-major order ($xstride < ystride < zstride$) is faster for some routines but not required [S3].
4.  **Memory Management:** The caller is responsible for freeing the `voxels` memory. VolPack does not free it upon context destruction [S3].

#### Variant C: 16-bit to 8-bit VDB Conversion (S4)
1.  **Parse Header:**
    *   Read resolution from offsets 0, 2, 4 as `Uint16`.
    *   Round each dimension to the nearest multiple of 8: $R' = 8 \cdot \lceil R / 8 \rceil$. This aligns with 8x8x8 voxel tiles [S4, unofficial].
    *   Read node count from offset 12 as `Uint32` [S4, unofficial].
2.  **Parse Coordinates:** Read coordinates from offset 32 as `Int32Array`. Cast to `Uint16Array` for storage [S4, unofficial].
    *   *Pitfall:* If coordinate values exceed 65535, they wrap around or clamp, corrupting sparse coordinate data [S4, unofficial].
3.  **Parse Node Data:** Read node data following coordinates as `Uint16Array` [S4, unofficial].
4.  **Compress Data:** Allocate a `Uint8Array` (`compressedData`) of the same length as `nodeData`.
5.  **Quantize Loop:** For each index $i$:
    $$
    compressedData[i] = \begin{cases} 
    0 & \text{if } nodeData[i] == 0 \\
    \lfloor 255 \cdot \frac{nodeData[i]}{65535} \rfloor & \text{otherwise}
    \end{cases}
    $$
    This maps the full 16-bit range $[0, 65535]$ to 8-bit $[0, 255]$ [S4, unofficial].

### Parameters
*   **Threshold:** 160 (for S1/S2) [S1, unofficial].
*   **Quantization Divisor:** 4096 (for S1/S2) [S1, unofficial].
*   **Tile Alignment:** 8 (for S4) [S4, unofficial].
*   **File Extension:** ".zaz" (for S1) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Coordinate Overflow:** In Variant C, casting 32-bit coordinates to 16-bit storage causes data corruption for large volumes [S4, unofficial].
*   **Memory Leaks:** In Variant B, failing to free the voxel array manually leads to leaks, as VolPack does not manage this memory [S3].
*   **Precision Loss:** Variant A discards the lower 8 bits of the 12-bit Hounsfield data. This is acceptable for visualization but loses fine-grained density information [S1, S2, unofficial].

## Subtopics

### [HDF5 JSON (H5J) File Parsing](2922-hdf5-json-h5j-file-parsing.md)
Parses HDF5 JSON (H5J) file attributes to extract volume size, voxel size, and channel specifications. It applies word alignment to volume dimensions. Unlike the raw binary loaders, this method handles structured metadata extraction from a hierarchical format.

### VolPack Raw Voxel Data Setup
A procedure to specify volume data array, size, and strides for each dimension in bytes using `vpSetRawVoxels`. It differs from direct array loading by using a library context and stride-based access. It is chosen when using the VolPack library for rendering or processing.(Parent)

### 16-bit to 8-bit Data Conversion
Converts 16-bit VDB data to 8-bit format for GPU consumption. It rounds resolution to multiples of 8 and compresses coordinates and density. It is chosen when preparing sparse VDB data for WebGL or other 8-bit constrained renderers.(Parent)

## Operating characteristics
*   **Time Complexity:** Linear $O(N)$ with respect to the number of voxels for loading and quantization [S1, S4, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the raw data. Variant B (VolPack) requires additional memory for the library context but shares the voxel data pointer [S3].
*   **Performance:** Z-major storage (Variant B) offers faster access for certain VolPack routines [S3]. Quantization (Variant A) reduces memory bandwidth requirements by lowering precision [S1, unofficial].
*   **Limitations:** Variant A is specific to 12-bit Hounsfield data. Variant C is limited by 16-bit coordinate storage [S4, unofficial].

## Background
The method originates from implementations of volume raycasting, specifically for the Stanford Bunny dataset and CT scans. The Stanford Bunny loader (S1, S2) was developed to visualize 12-bit Hounsfield data from CT scanners using OpenCL. VolPack (S3) is a standard library for volume rendering, providing a generic interface for raw voxel data. The VDB conversion (S4) is part of modern web-based volume rendering pipelines (e.g., volumetrics) to optimize data transfer to GPUs.

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S3] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S4] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
