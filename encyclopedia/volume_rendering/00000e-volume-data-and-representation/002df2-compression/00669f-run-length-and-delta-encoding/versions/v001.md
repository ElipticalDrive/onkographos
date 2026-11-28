# Run-Length and Delta Encoding

Also called: Delta RLE, Delta Run-Length Encoding, ΔRLE (Delta Run-Length Encoding)

## Overview
Run-Length and Delta Encoding (ΔRLE) is a lossless compression algorithm for volumetric data that processes input as a stream of bits divided into blocks, selecting the most efficient representation from four types: Raw, RLE, Delta-Transformed RLE, or Repeating, to minimize total output size while bounding maximum inflation [S1]. It differs from standard [Run-Length Encoding for Voxel Data](5051-run-length-encoding-for-voxel-data.md) by operating on fixed-size bit blocks with delta encoding (XOR with the previous block) and optimized, dissociated variable-length encoding words, and from [Sparse structures](5-sparse-structures.md) by maintaining a linear stream structure rather than hierarchical spatial subdivisions [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A stream of original data bits, parameter $m$ (overall length of input stream in bits), and parameter $n$ (length of Original Data Block (ODB) in bits) [S1].
- **Output:** A compressed bitstream consisting of a Global Head (GH) followed by a sequence of Compressed Data Blocks (CDB) [S1].

**Data Structures:**
The compressed output consists of the following structures:

1.  **Global Head (GH):**
    -   **GLobal Size Packet (GLSP):** One or more 32-bit Encoding Words (EW) summing to the total source bit count $s$. An EW with value $< 2^{32}-1$ indicates another EW follows; value $2^{32}-1$ terminates the packet [S1].
    -   **Data Block Size Packet (DBSP):** One or more 16-bit EWs summing to the bit count of the Original Data Block (ODB). Max value is $2^{16}-1$ [S1].

2.  **Compressed Data Block (CDB) Types:** Each CDB has an 11-bit header (except RawDB and RepDB) followed by payload.
    -   **Raw Data Block (RawDB):**
        -   Header (RawDBH): 2 bits `'00'`.
        -   Payload: $n$ bits of uncompressed ODB [S1].
    -   **RLE Data Block (RleDB):**
        -   Header (RleDBH): 11 bits.
            -   Bits 0-1: `'01'` (Identifier).
            -   Bit 2: $S$ (Symbol selector; if $S=0$, even RLE packets encode runs of 0s; if $S=1$, even RLE packets encode runs of 1s) [S1].
            -   Bits 3-6: `0WW` (4 bits, length of EWs for symbol 0 runs, range $<2, 15>$) [S1].
            -   Bits 7-10: `1WW` (4 bits, length of EWs for symbol 1 runs, range $<2, 15>$) [S1].
        -   Payload: Stream of RLE Packets (RLEPs) [S1].
    -   **Delta-transformed Data Block (DelDB):**
        -   Header (DelDBH): 11 bits.
            -   Bits 0-1: `'10'` (Identifier).
            -   Bit 2: $S$ (Symbol selector, same logic as RleDBH) [S1].
            -   Bits 3-6: `0WW` (4 bits) [S1].
            -   Bits 7-10: `1WW` (4 bits) [S1].
        -   Payload: Stream of RLE Packets encoding the Delta Data Block (DDB) [S1].
    -   **Repeating Data Block (RepDB):**
        -   Header (RepDBH): 2 bits `'11'` (Identifier) [S1].
        -   Payload: One or more 3-bit EWs. Max value for 3-bit EW is 7. An EW with value $<7$ indicates another EW follows [S1].

3.  **RLE Packet (RLEP):**
    -   Composed of one or more Encoding Words (EW).
    -   EW length is defined by `0WW` (for symbol 0) or `1WW` (for symbol 1).
    -   Max value for EW with length $l$ is $\text{max} = 2^l - 1$.
    -   Run length is the sum of the values of concatenated EWs. An EW with value $< \text{max}$ indicates another EW follows [S1].

**Algorithm Steps:**
The compression algorithm processes the input stream block by block.

1.  Initialize Previous Data Block $PDB(n-1:0) = 0$ and Repetition counter $REPS = 0$ [S1].
2.  Load the next Actual Data Block $ADB(n-1:0)$ from the input stream [S1].
3.  If $ADB == PDB$, increment $REPS$ and return to Step 2 (process next block) [S1].
4.  If $REPS > 0$, create a RepDB encoding the count $REPS$, store it, and reset $REPS = 0$ [S1].
5.  Set minimum bit cost $min = n + 2$ (cost of RawDB) [S1].
6.  Call `findMin(ADB, 0, 2, 15)` to get `minRleDB0` (min bits for symbol 0) and `Rle0WL` (optimal EW length for symbol 0) [S1].
7.  Call `findMin(ADB, 1, 2, 15)` to get `minRleDB1` (min bits for symbol 1) and `Rle1WL` (optimal EW length for symbol 1) [S1].
8.  Calculate total RLE cost: `minRleDB = minRleDB0 + minRleDB1` [S1].
9.  Compute Delta Data Block $DDB = PDB \oplus ADB$ (bitwise XOR) [S1].
10. Call `findMin(DDB, 0, 2, 15)` to get `minDelDB0` and `Del0WL` [S1].
11. Call `findMin(DDB, 1, 2, 15)` to get `minDelDB1` and `Del1WL` [S1].
12. Calculate total Delta cost: `minDelDB = minDelDB0 + minDelDB1` [S1].
13. If `minRleDB < min` AND `minRleDB <= minDelDB`, create `RleDB(ADB, Rle0WL, Rle1WL)`, store it, and jump to Step 16 [S1].
14. If `minDelDB < min` AND `minDelDB < minRleDB`, create `DelDB(DDB, Del0WL, Del1WL)`, store it, and jump to Step 16 [S1].
15. Otherwise, create `RawDB`, store it [S1].
16. If $m == 0$, exit algorithm [S1].
17. Update $PDB = ADB$ [S1].
18. If $m > n$, decrement $m$ by $n$. Else, set $n = m$ and $m = 0$ [S1].
19. Return to Step 2 [S1].

**Helper Function `findMin(D, S, L, H)`:**
-   Finds the minimal sum of bits for all RLE packets for symbol $S$ in data block $D$, testing all Encoding Word Lengths from range $<L, H>$ [S1].
-   Returns minimal bit count (a) and optimal EW length (b) [S1].
-   Implementation logic: Iterate $l$ from $L$ to $H$. For each $l$, calculate the total bits required to encode all runs of symbol $S$ in $D$ using EWs of length $l$. The total bits for a run of length $R$ with EW length $l$ is calculated by decomposing $R$ into a sequence of EWs where each EW value $v < 2^l-1$ contributes $l$ bits, and the final EW value $v_{final} = 2^l-1$ contributes $l$ bits. Sum these bits for all runs. Return the $l$ that minimizes this sum [S1, G].

**Formulas:**
-   **Length of RepDB:** $L_{RepDB} = 2 + \lceil \frac{h}{7} \rceil \times 3$ bits, where $h$ is the number of repetitions [S1].
-   **Maximum Output Size (Worst Case):** $S_{max} = 32 \times \lceil \frac{32}{2^1 - 1} \rceil + 16 \times \lceil \frac{16}{2^1 - 1} \rceil + m \times (n + 2)$ bits. The first two terms represent overhead for RLE headers/structures, and the last term is the sum of RawDB sizes for $m$ blocks of $n$ bits [S1].
-   **Minimum Output Size (Best Case, homogeneous 0s):** $S_{min} = 32 \times \lceil \frac{32}{2^1 - 1} \rceil + 16 \times \lceil \frac{16}{2^1 - 1} \rceil + 2 + 3 \times \lceil \frac{m}{2^1 - 1} \rceil$ bits. This assumes a single RepDB covers all $m$ blocks [S1].
-   **Maximum Inflation:** $I_{max} = \frac{n + 2}{n}$ [S1].

**Parameters:**
-   $n$: Length of Original Data Block (ODB) in bits. Typical values in tests: 8, 16. Larger $n$ reduces maximum inflation [S1].
-   $m$: Overall length of the input stream in bits. Used to track remaining data for termination [S1].

**Edge Cases and Pitfalls:**
-   **Data Inflation:** If data cannot be compressed effectively, RawDB is used, adding a 2-bit header per block. The maximum inflation is bounded by $(n+2)/n$. For small $n$, this overhead is significant [S1].
-   **Bit Alignment:** The binary representation of ΔRLE compressed data is not byte-aligned, requiring careful bit-level handling during read/write operations [S1].
-   **Computational Cost:** Analyzing optimal encoding word lengths is computationally expensive, causing worst-case compression times up to 3.50× longer than RLE [S1].

## Subtopics

### [Auto-trigger RLE with Variable Length Codes](3382-auto-trigger-rle-with-variable-length-codes.md)
Auto-trigger RLE with Variable Length Codes is a lossless compression method for voxel data that eliminates redundancy by encoding runs of identical voxels using variable-length codes (VLCs) triggered only after a specific repetition threshold. It differs from standard "all-counts" RLE by avoiding the overhead of encoding short, non-repeating sequences as zero-length runs, and from fixed-length RLE by adapting to the skewed statistical distribution of voxel types and run lengths.

### [Run-Length Encoding for Voxel Data](5051-run-length-encoding-for-voxel-data.md)
Run-Length Encoding (RLE) is a lossless compression technique that represents consecutive uniform voxels—particularly empty or homogeneous runs—as compact lists of start positions, lengths, and values, enabling rays to skip entire homogeneous regions in a single operation. It differs from sibling methods like [Sparse structures](5-sparse-structures.md) by operating on linear data streams rather than hierarchical spatial subdivisions, and from [Arithmetic Encoding](2127-arithmetic-encoding.md) by using fixed-length count/value pairs instead of variable-length entropy codes, which facilitates faster, deterministic decoding suitable for real-time ray marching.

### Block Delta Encoding with LZO
Block Delta Encoding with LZO performs delta encoding of block data with respect to the corresponding block of the previous time step. Each block is quantized with a proper bit count per voxel depending on the range of data inside the brick. Finally, all blocks are encoded with a LZO algorithm [S2].

### Column Delta Encoding
Column Delta Encoding is a compression technique where reference columns are stored fully, and subsequent columns are stored as deltas (differences) from a reference column. It is effective when terrain generation creates similar adjacent columns [S3, unofficial]. Effectiveness depends heavily on the specifics of the data; it works well for terrain generation with similar adjacent columns but results may vary (YMMV) [S3, unofficial]. It provides about 4x compression beyond what is achieved from RLE alone [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
-   **Compression Time:** ΔRLE is generally slower than classical RLE. For the 'Head' dataset, RLE took 0.017s while ΔRLE took 0.051s (ratio 3.06). The time ratio (ΔRLE/RLE) ranges from 0.43 (best case with consecutive repeating blocks) to 3.50 (worst case) [S1].
-   **Decompression Time:** ΔRLE decompression is slower than RLE, with a time ratio ranging from 1.73 (best case) to 2.66 (worst case) [S1].
-   **Memory:** ΔRLE compressor requires 57B for variables, 2×$n$ bits for two data blocks, 32B Look-Up-Table, and 64B for encoding word length optimization. ΔRLE decompressor requires 33B for variables, 2×$n$ bits for two data blocks, and 32B Look-Up-Table [S1].

**Measured Performance:**
-   Tested on medical imaging volumes (CT/MRI) thresholded to binary. ΔRLE achieved compression ratios of 3.972 to 48.911 compared to original thresholded data, and 1.24 to 3.44 times higher compression ratios than classical RLE (RLE-I) [S1].
-   For the 'Head' dataset (256x256x113, 16b/vox), ΔRLE compressed data size was 628,533 bits, achieving a compression ratio of 11.78 [S1].
-   For the 'Pancreas' dataset (240x512x512, 16b/vox), ΔRLE used 220,245 RepDBs (84.02% of blocks), 41,775 RleDBs (15.93%), and 124 DelDBs (0.05%). No RawDBs were used [S1].

**Strengths and Limitations:**
-   **Strengths:** Higher compression ratios than classical RLE in all tested cases by selecting the optimal block type and variable encoding word lengths [S1]. Bounded maximum inflation [S1].
-   **Limitations:** Higher computational overhead for length optimization [S1]. Bit-level handling required for non-byte-aligned data [S1].

**When to Use:**
-   Use when high compression ratios are needed for volumetric data with spatial redundancy or temporal coherence (via delta encoding), and computational cost is acceptable.
-   Do not use when real-time decompression speed is critical and data is not highly redundant, as ΔRLE decompression is slower than RLE [S1].

## Background
ΔRLE was introduced by Madoš, Bilanová, and Hurtuk in 2021 as a lossless data compression algorithm for volumetric datasets [S1]. It enhances bit-level RLE by optimizing RLE packets with multi-encoding word packets and variable encoding word lengths, and by incorporating delta transformation and repeating block detection [S1]. It is used in the context of volume data representation, particularly for medical imaging data [S1].

## Sources
[S1] ΔRLE, https://doi.org/10.31341/jios.45.1.15, fetched 2026-10-27
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] Method to compress chunks for voxel engine?, https://www.reddit.com/r/VoxelGameDev/comments/15bycu3/method_to_compress_chunks_for_voxel_engine/, fetched 2026-10-27 (unofficial)
