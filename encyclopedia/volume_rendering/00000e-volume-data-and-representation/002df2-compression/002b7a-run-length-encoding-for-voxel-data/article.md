# Run-Length Encoding for Voxel Data

Also called: RLE, RLE Voxel Compression, Run Length Coding, Run-Length Encoding, 行程编码

## Overview
Run-Length Encoding (RLE) is a lossless compression technique that represents consecutive uniform voxels—particularly empty runs—as compact lists of start positions, lengths, and values [S1]. It solves the problem of high memory bandwidth and traversal overhead in sparse volumetric data by enabling rays to skip entire homogeneous regions in a single operation [S1]. Unlike general-purpose entropy coders like Arithmetic Encoding [2127-arithmetic-encoding.md] which require sequential decoding, RLE provides a structure where transparent sequences can be jumped over directly [G]. It is distinct from sibling methods like [Sparse structures](5-sparse-structures.md) that rely on tree hierarchies or [Deferred Filtering Architecture for Compressed Volumes](857-deferred-filtering-architecture-for-compressed-volumes.md) which use block-based filtering [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A one-dimensional array or stream of voxel values (e.g., `uint8` or `uint16` density/occupancy values) [S6, unofficial].
*   **Output:** A compressed array of alternating `(count, value)` pairs [S2].

**Data Structures:**
The compressed data is stored as a linear array where elements alternate between run length and run value [S2].
*   **Standard Types (Short/Int/Byte):** The array consists of pairs of elements of the same type [S3, unofficial]. For example, if the input is `uint16`, the output is a `Uint16Array` where index $2i$ is the count and $2i+1$ is the value [S6, unofficial].
*   **Boolean Type:** A specialized layout uses a single signed byte per run [S3, unofficial]. The absolute value represents the count, and the sign bit indicates the boolean value (negative for `true`, positive for `false`) [S3, unofficial].

**Parameters:**
*   **Count Limit:** The maximum run length is constrained by the data type of the count field [S3, unofficial].
    *   Byte RLE: Max count 127 (`Byte.MAX_VALUE`) [S3, unofficial].
    *   Short RLE: Max count 32,767 (`Short.MAX_VALUE`) [S3, unofficial].
    *   Int RLE: Max count 2,147,483,647 (`Integer.MAX_VALUE`) [S3, unofficial].
*   **Voxel Data:** Typically `uint8` (0-255) for density or material ID [S6, unofficial].

**Algorithm (Encoding and Decoding):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_rle(voxel_data):
    """Compress a 1D voxel stream into (count, value) pairs."""
    if not voxel_data:
        return []
    compressed = []
    count = 1
    current_value = voxel_data[0]
    for i in range(1, len(voxel_data)):
        if voxel_data[i] == current_value:
            count += 1
            if count == 32767:  # Short RLE limit [S3, unofficial]
                compressed.append((count, current_value))
                count = 1
                current_value = voxel_data[i]
        else:
            compressed.append((count, current_value))
            count = 1
            current_value = voxel_data[i]
    compressed.append((count, current_value))
    return compressed

def decode_rle(compressed, sz):
    """Reconstruct the original voxel stream from (count, value) pairs."""
    decompressed = [0] * sz  # Initialize empty output array [G]
    idx = 0
    for count, value in compressed:
        for _ in range(count):
            decompressed[idx] = value
            idx += 1
    return decompressed
```

**Ray Marching Integration:**
During rendering, the ray traverses the encoded runs [S1]. If the run's value indicates transparency (e.g., 0 or background), the ray advances its position by the run's length (scaled by voxel size) and skips to the next run [S1]. This continues until an opaque run is encountered or the volume boundary is reached [S1].

**Edge Cases and Pitfalls:**
*   **Count Overflow:** If a run exceeds the maximum value of the count type (e.g., >32767 for Short RLE), the encoder must split the run into multiple entries [S3, unofficial].
*   **Data Differencing:** Applying data differencing (predicting the next value) before RLE does not improve compression for voxel data and may degrade it [S7, unofficial].
*   **GPU Random Access:** RLE is not suitable for GPU random access or arbitrary 3D edits because decoding requires sequential traversal, making it difficult to update individual voxels without decompressing the entire run [S8, unofficial].

## Subtopics

### [Chunk Serialization with RLE](5162-chunk-serialization-with-rle.md)
Chunk Serialization with RLE compresses voxel chunk data by storing it in RLE format when idle, decompressing it on first write access, and freeing memory if the chunk becomes empty. It achieves a 58x size reduction compared to raw storage (39.06 MB to 0.67 MB). This method is distinct from static RLE storage by introducing lazy decompression and memory management based on access patterns.

### [VOXLAP RLE Voxel Storage](5172-voxlap-rle-voxel-storage.md)
VOXLAP RLE Voxel Storage is an in-memory compressed voxel data structure used by the VOXLAP engine for CPU-based volume rendering. It solves the problem of efficiently storing and editing sparse voxel grids using a Run-Length Encoding (RLE) scheme organized as a 2D array of columns. It sets itself apart from generic RLE methods by its specific column-oriented arena allocation and batched edit support, tailored for the constraints of the original VOXLAP engine.

### [Window-Length Encoding](6788-window-length-encoding.md)
Window-Length Encoding is a compression technique applied to intermediate images in distributed volume rendering to reduce network transfer time. It differs from sibling methods by specifically targeting the pixel data of rendered intermediate images rather than raw voxel data, leveraging the spatial coherence of distant views to achieve significant bandwidth savings.

### [DCx Voxel Configuration Encoding](9332-dcx-voxel-configuration-encoding.md)
DCx Voxel Configuration Encoding is a method for compactly representing the occupancy patterns of an "expanded cube" (a group of 8 voxels) within the Dual Contouring over Expanded Cubes (DCx) framework. It solves the problem of efficiently indexing and storing the 256 possible binary occupancy states by grouping rotationally and mirror-symmetric patterns into equivalence classes. It differs from its parent [Run-Length Encoding for Voxel Data](5051-run-length-encoding-for-voxel-data.md) by focusing on topological equivalence classes for mesh generation rather than run-length compression of raw data, and from siblings like [Chunk Serialization with RLE](5162-chunk-serialization-with-rle.md) by using a fixed 17-category lookup table based on geometric symmetry rather than variable-length run counts.

### Run-length and palette-compressed bricks
Voxlap's VXL format stores each 1024×1024 column as spans and keeps colour only for "surface voxels" (solid voxels with an air neighbour), making interior solid implicit [S8, unofficial]. Minecraft stores each 16×16×16 section as a palette plus packed indices of at least 4 bits (up to 12); a section with one block state stores no index array at all [S8, unofficial]. This is the standard, easy way to compress materials, but RLE does not suit GPU random access and arbitrary 3D edits [S8, unofficial].

### Y-axis Voxel Column Compression
This method compresses voxel chunk data by storing only the lowest Y coordinate for each contiguous block ID along each X-Z column, reducing bandwidth for transmission [S9].
**Steps:**
1.  Iterate over each X and Z coordinate in the chunk [S9].
2.  Initialize `LowestSimilarBlockColor` to `nil` [S9].
3.  Loop through Y coordinates from 1 to the column height [S9].
4.  If the current block color differs from `LowestSimilarBlockColor`, update `LowestSimilarBlockColor` to the current color and store `compressedgrid[x][z][y] = color` [S9].
5.  After the Y loop, if `grid[x][z][y + 1] == nil`, set `compressedgrid[x][z][y + 1] = 0` to mark the top of the column as an air block and prevent decompression table overflow [S9].
6.  During decompression, if `color == 0`, skip the entry to ignore air blocks [S9].
**Pitfalls:** Using non-deterministic iteration (e.g., Lua's `pairs()`) causes voids or missing blocks [S9]. The decompression loop relies on the next Y coordinate being explicitly stored as a non-nil value to terminate; missing boundary markers cause infinite loops or table overflows [S9].
**Performance:** Reduces a 16x32x16 voxel chunk from 16.25KB (JSON encoded) to under 2KB [S9].

### Run-Length Encoding for Material Indices
A lossless entropy encoding technique applied to numerical material index arrays to exploit data repetition, specifically uniform sequences of solid and empty material indices [S10, unofficial].
**Steps:**
1.  Identify a streak of repeating values (a run) in the one-dimensional material index array [S10, unofficial].
2.  Count the number of occurrences in the run (run-length) [S10, unofficial].
3.  Replace the $n$ occurrences of data item $d$ with the single pair $(d, n)$ [S10, unofficial].
4.  Store run-lengths as 32-bit unsigned integers to accommodate the maximum number of material indices [S10, unofficial].
5.  If a set of Hermite data contains only solid material indices (full set), output only one material index and one run-length holding the total count [S10, unofficial].
6.  Discard empty sets as they contain no meaningful information [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N)$ where $N$ is the number of voxels. Decoding is $O(M)$ where $M$ is the number of runs. Ray traversal is proportional to the number of opaque runs intersected, offering significant speedups for sparse data [S1].
*   **Memory Complexity:** The compressed size is proportional to the number of runs. In highly homogeneous data (e.g., empty space), the size is $O(1)$ per large region. In worst-case alternating data, the size is $2N$ (count + value for each voxel), which is worse than raw storage [G].
*   **Measured Performance:** In simulated fluid dynamics volumes with long empty stretches, compression ratios exceed 10:1 and rendering times are reduced by 5-15 times relative to naive traversal [S1].
*   **Strengths:** Extremely fast decoding for sequential access; ideal for sparse volumes with large empty regions; simple hardware implementation [S1].
*   **Limitations:** Not suitable for random access or in-place editing on GPUs due to sequential dependency [S8, unofficial].
*   **When to use:** Sparse volumes, terrain data, or datasets with large homogeneous regions (e.g., air/background) [S1].
*   **When not to use:** Dense, noisy data; scenarios requiring frequent random access or partial updates [S8, unofficial].

## Background
RLE was widely used in compressed Direct Volume Rendering (DVR) since early applications [S3, unofficial][S4]. It is frequently combined with entropy encoders (e.g., Huffman or Arithmetic coding) to accelerate decoding or reduce on-disk storage [S3, unofficial][S4]. Hardware implementations, such as the VolumePro ASIC, integrated RLE directly into the ray-casting pipeline for real-time performance on commodity systems [S1].

## Sources
[S1] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S2] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27
[S3] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27 (unofficial)
[S4] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S5] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27
[S6] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S7] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S8] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28 (unofficial)
[S9] Voxel Data Compression - Scripting Support, https://devforum.roblox.com/t/voxel-data-compression/1875040, fetched 2026-10-27
[S10] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26 (unofficial)
