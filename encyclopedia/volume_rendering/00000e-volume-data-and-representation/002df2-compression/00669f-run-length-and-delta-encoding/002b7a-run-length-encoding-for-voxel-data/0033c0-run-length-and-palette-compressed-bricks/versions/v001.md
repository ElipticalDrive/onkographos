# Run-length and palette-compressed bricks

Also called: Minecraft-style palette compression, Palette-compressed bricks

## Overview
This method compresses voxel data by grouping voxels into fixed-size bricks (typically $16^3$) and storing a small palette of material IDs alongside packed indices that reference these materials. It solves the problem of high memory usage for large-scale voxel worlds with many repeated material types, offering a standard, easy way to compress materials [S1]. Unlike pure run-length encoding, which struggles with GPU random access and arbitrary 3D edits, palette compression allows for efficient random access to individual voxel types within a brick [S1].

## How it works
**Inputs:** A 3D voxel grid where each voxel has a material ID (uint8 or uint16).
**Outputs:** A compressed representation consisting of a palette array and a packed index array per brick.

**Data Structures:**
1.  **Palette:** An array of unique material IDs present in the brick. Size $N_p \le 2^k$, where $k$ is the bit-depth of the index.
2.  **Index Array:** A bit-packed array storing indices into the palette. Each voxel in the brick is represented by $k$ bits. For a $16 \times 16 \times 16$ brick, there are 4096 voxels.
    *   If $N_p \le 16$, $k=4$ bits.
    *   If $N_p \le 256$, $k=8$ bits.
    *   If $N_p \le 4096$, $k=12$ bits.
    *   *Note:* The source specifies "at least 4 bits (up to 12)" [S1].

**Steps:**
1.  **Partition:** Divide the voxel volume into non-overlapping bricks of size $B \times B \times B$ (standard $16^3$) [S1].
2.  **Identify Unique Materials:** For each brick, collect all unique material IDs present in the voxels.
3.  **Build Palette:** Create the palette array from the unique IDs. Let the size be $N_p$.
4.  **Determine Bit Depth:** Calculate $k = \lceil \log_2(N_p) \rceil$. If $N_p=1$, $k=0$ (no index array needed) [S1].
5.  **Map Indices:** Replace each voxel's material ID with its index in the palette.
6.  **Pack Indices:** Pack the $k$-bit indices into a contiguous byte array. For $k=4$, two indices fit in one byte. For $k=8$, one index per byte. For $k=12$, three indices fit in four bytes (with padding/alignment considerations) [S1].

**Parameters:**
*   Brick size: $16 \times 16 \times 16$ [S1].
*   Palette size limit: Implicitly determined by available bit depths (4, 8, 12 bits) [S1].

**Edge Cases:**
*   **Single Material:** If a brick contains only one material type, the index array is omitted entirely to save space [S1].
*   **Palette Overflow:** If more than $2^{12}$ unique materials exist in a $16^3$ brick, this method as described cannot represent it without increasing bit depth or splitting the brick. Not given in the sources.

## Subtopics

### Palette Encoding
Retrieves the palette ID for a texel index given a pattern number. The process involves decomposing the pattern index $p$ into $p_2 = p/2$ and $p_3 = p\%2$. The encoded value is selected from a switch-like structure using bitmasks based on $p_2$. If $p_3$ is 1, the result is shifted right by 16 bits. Finally, bit $i$ is extracted [S2, unofficial].

## Operating characteristics
*   **Compression Ratio:** High for scenes with repetitive materials. A brick with 1 unique material uses 0 bits for indices. A brick with 16 materials uses 4 bits per voxel (512 bytes for 4096 voxels vs 4096 bytes uncompressed).
*   **Random Access:** Supports efficient random access to voxel material types within a brick, unlike RLE which requires sequential traversal [S1].
*   **Editability:** Suitable for arbitrary 3D edits, unlike RLE which is poor for GPU random access and edits [S1].
*   **Complexity:** Encoding is $O(V)$ where $V$ is the number of voxels in the brick. Decoding is $O(1)$ per voxel with bit-shifting.
*   **Limitations:** Palette size is limited by bit depth. Not suitable for continuous data (e.g., density fields) without quantization [S1].

## Background
This approach is exemplified by Minecraft's chunk format, which stores $16 \times 16 \times 16$ sections as a palette plus packed indices [S1]. It is described as the standard, easy way to compress materials in voxel engines [S1]. Voxlap's VXL format uses a similar concept for columns, storing spans and keeping color only for surface voxels, though it uses RLE for spans rather than palette compression for the entire column [S1].

## Sources
[S1] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
[S2] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
