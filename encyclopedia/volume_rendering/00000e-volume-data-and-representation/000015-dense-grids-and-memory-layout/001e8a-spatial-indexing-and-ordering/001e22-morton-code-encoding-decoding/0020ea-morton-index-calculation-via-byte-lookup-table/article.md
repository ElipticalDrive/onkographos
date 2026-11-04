# Morton Index Calculation via Byte-Lookup Table

Also called: Morton Code Utility, Morton encoding with LUT

## Overview
A method to compute the Morton (Z-order) curve index for a 3D voxel position $(x, y, z)$ by using precomputed lookup tables for each byte of the coordinates, avoiding expensive bitwise interleaving operations [S2, unofficial]. It sets itself apart from naive bitwise interleaving by trading memory access for computational simplicity, and from general Morton encoding by optimizing specifically for chunked voxel data with a hard limit of $256^3$ voxels per chunk [S2, unofficial]. It builds on [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md).

## How it works
**Inputs and Outputs:**
- Input: 3D integer coordinates $(x, y, z)$.
- Output: A Morton code (integer) representing the interleaved position.

**Data Structures:**
The method relies on three lookup tables, `morton256_x`, `morton256_y`, and `morton256_z`.
- Each table contains 256 entries.
- Each entry maps a single byte (0–255) to its 3-bit Morton-encoded representation.
- The 3-bit representation interleaves the input byte's bits with zeros for the other two dimensions. For example, `morton256_x[b]` places the bits of byte `b` into the X-bit positions of the 3D Morton code, with Y and Z bits set to 0.
- In the `libmorton` implementation, these are accessed via `libmorton::m3D_e_sLUT` [S1, unofficial].
- The `morton_type` is `uint16_t` for 16-bit width or `uint32_t` for 32-bit width [S1, unofficial].
- The `component_type` is `uint8_t` for 16-bit width or `uint16_t` for 32-bit width [S1, unofficial].

**Algorithm Steps:**
1. **General Case (Large Coordinates):** If coordinates exceed 255, the input is processed byte-by-byte from most significant to least significant.
   - Extract the most significant byte of $x, y, z$ using shift and mask operations.
   - Look up the encoded byte in the respective tables.
   - Combine results using bitwise OR and shift operations to build the final index.
   - Repeat for subsequent bytes [S2, unofficial].
   
2. **Optimized Case (Chunked Data $\le 256^3$):** For chunks limited to $256^3$ voxels, coordinates fit within 8 bits per axis.
   - Directly index the 256-entry tables with the full 8-bit coordinate components.
   - Combine the three results using bitwise OR.
   - Formula:
     $$ \text{index} = \text{morton256\_x}[x] \mid \text{morton256\_y}[y] \mid \text{morton256\_z}[z] $$
   - This avoids shifts and masks for byte extraction [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed tables: morton256_x, morton256_y, morton256_z
# Each is an array of 256 integers.
# morton256_x[b] contains the bits of b interleaved into X positions.

def encode_morton_lut_optimized(x: int, y: int, z: int) -> int:
    """
    Encodes 3D coordinates into a Morton index using byte-lookup tables.
    Assumes x, y, z are < 256.
    """
    # Direct lookup and bitwise OR
    index = morton256_x[x] | morton256_y[y] | morton256_z[z]
    return index

def encode_morton_lut_general(x: int, y: int, z: int) -> int:
    """
    Encodes 3D coordinates into a Morton index using byte-lookup tables.
    Handles coordinates up to 32-bit integers by processing bytes.
    """
    answer = 0
    
    # Process most significant byte (assuming 32-bit input, byte 2)
    # Shift right by 16 bits to get the third byte (bits 16-23)
    # Mask with 0xFF to isolate the byte
    b_x = (x >> 16) & 0xFF
    b_y = (y >> 16) & 0xFF
    b_z = (z >> 16) & 0xFF
    
    # Look up and combine
    # Start with the most significant part of the result
    answer = (morton256_z[b_z] | morton256_y[b_y] | morton256_x[b_x])
    
    # Shift left to make room for the next byte
    answer = answer << 24
    
    # Process second byte (bits 8-15)
    b_x = (x >> 8) & 0xFF
    b_y = (y >> 8) & 0xFF
    b_z = (z >> 8) & 0xFF
    
    answer = answer | morton256_z[b_z] | morton256_y[b_y] | morton256_x[b_x]
    answer = answer << 24
    
    # Process least significant byte (bits 0-7)
    b_x = x & 0xFF
    b_y = y & 0xFF
    b_z = z & 0xFF
    
    answer = answer | morton256_z[b_z] | morton256_y[b_y] | morton256_x[b_x]
    
    return answer
```

**Parameters:**
- `AXIS_MAX`: Defines the maximum coordinate value per axis. 32 for 16-bit width, 1024 for 32-bit width [S1, unofficial].
- Table Size: 256 entries per table [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Size:** Creating a lookup table for every possible unsigned int value is impractical due to memory size; therefore, the table is limited to 256 entries (one byte) and applied repeatedly [S2, unofficial].
- **Coordinate Bounds:** The optimized 3-lookup version assumes coordinates are $< 256$. Using it with larger coordinates will produce incorrect results [S2, unofficial].
- **Precision:** The method uses integer arithmetic. Ensure the output type (`morton_type`) is large enough to hold the interleaved bits without overflow [S1, unofficial].

## Subtopics

### [Fast Neighbourhood Access via Delta Lookup Tables](3652-fast-neighbourhood-access-via-delta-lookup-tables.md)
A method to access immediate neighbours of a voxel in Morton-ordered memory without re-encoding coordinates, using precomputed offset tables that map a coordinate component to the memory stride required to reach its neighbour. It solves the performance bottleneck of decoding a Morton index to $(x,y,z)$, modifying coordinates, and re-encoding, which involves expensive bitwise operations. It is significantly faster than the naive decode-modify-encode approach and offers a simpler implementation alternative to bitwise manipulation methods like Fabian Giesen’s. See [Fast Neighbourhood Access via Delta Lookup Tables](3652-fast-neighbourhood-access-via-delta-lookup-tables.md).

### Sparse Voxel Octree Morton Code Encoding/Decoding
Encodes/decodes a list of sorted sparse voxel Morton codes into a compact uint8 sparse voxel octree representation, with optional parent-delta attribute encoding. It takes a tensor of $N$ sorted uint32 Morton codes and a depth parameter, and optionally attributes for delta encoding. See [Sparse Voxel Octree Morton Code Encoding/Decoding](3652-sparse-voxel-octree-morton-code-encoding-decoding.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ for the optimized chunked case (3 lookups, 2 ORs). $O(\log N)$ for the general case where $N$ is the number of bits in the coordinate (typically 32, so 3-4 lookups and shifts).
- **Memory Complexity:** $O(1)$ auxiliary memory for the lookup tables (3 tables $\times$ 256 entries).
- **Measured Performance:** Compared to linear indexing, computing Morton index via LUT takes ~40% longer [S2, unofficial]. Compared to bitwise operations, LUT is faster [S2, unofficial]. For chunk sizes $\le 256^3$, the optimized 3-lookup version is ~1-2% faster than linear indexing [S2, unofficial].
- **Strengths:** Fast neighbourhood access when combined with delta lookup tables; simple implementation; avoids expensive bitwise interleaving.
- **Limitations:** Requires precomputed tables; optimized version is limited to $256^3$ chunks; general version is still slower than linear indexing for simple access patterns.
- **When to Use:** When spatial locality is critical and neighbourhood access is frequent. When chunk sizes are small ($\le 256^3$).
- **When Not to Use:** When memory is extremely constrained and lookup tables cannot be afforded. When coordinates are very large and the general case overhead is unacceptable compared to bitwise methods.

## Background
The method originates from Jeroen Baert's extensive tests on the most performant way to determine Morton curve positions from a 3D input position, which concluded that lookup tables are faster than bitwise operations on modern hardware [S2, unofficial]. It was implemented in the context of Sparse Voxel Octree (SVO) rendering [S2, unofficial]. The `libmorton` library provides a template utility for this, supporting 16-bit and 32-bit Morton widths [S1, unofficial]. It is used in volume rendering and sparse voxel octree implementations [S1, unofficial][S2, unofficial].

## Sources
[S1] SeveriSuominen/rapid-svo, https://github.com/SeveriSuominen/rapid-svo, fetched 2026-10-26 (unofficial)
[S2] Implementing Morton ordering for chunked voxel data, http://www.volumesoffun.com/implementing-morton-ordering-for-chunked-voxel-data/index.html, fetched 2026-10-30 (unofficial)
[S3] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
