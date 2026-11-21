# Binvox File Parsing

Also called: Binvox File Parser, Binvox Header & RLE Voxel Reader, Binvox Header Parsing, Binvox SVO Import, Binvox to Schematic Conversion, binvox loader

## Overview
Binvox file parsing reads the binary voxel format, interpreting a fixed ASCII header to determine dimensions and metadata, then decompressing run-length encoded (RLE) voxel data into a contiguous memory structure [S1, S2, S3, unofficial]. It differs from general volume I/O methods by specifically handling the binvox format's unique combination of ASCII metadata and binary RLE payload, enabling efficient conversion into Sparse Voxel Octrees (SVO) or dense voxel grids [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary file or byte buffer containing a valid `.binvox` file.
- Output: A data structure representing the voxel grid (e.g., a contiguous `unsigned char` array, a Sparse Voxel Octree, or a Schematic object) and associated metadata (dimensions, translation, scale).

**Data Structures:**
1.  **Header State (S3):**
    - `mHeaderRead`: Boolean, initialized to `false`.
    - `mVoxelsRead`: Boolean, initialized to `false`.
    - `mDimensions`: Vector3 (X, Y, Z), parsed from the `dim` line.
    - `mTranslate`: Vector3, parsed from the `translate` line.
    - `mScale`: Double, parsed from the `scale` line.
    - `mVoxels`: Byte array, allocated with size $X \times Y \times Z$. Stores `1` for occupied voxels and `0` for empty.

2.  **SVO Node Structure (S2):**
    - `data.color`: 32-bit integer (e.g., `0xff00ffff` for internal nodes, `0` for empty leaves).
    - `occupancy`: 8-bit integer bitmask indicating which of the 8 children are occupied.
    - `children`: Array of 8 references to child nodes.

3.  **Flat Voxel Array (S2):**
    - `data`: Array of size $dim^3$ (where $dim$ is the dimension from the header), initialized to `0`.

**Numbered Steps:**

1.  **Initialize Reader:** Create a buffered reader for the input binary data [S2, unofficial].
2.  **Validate Magic Number:** Read the first line. If it does not start with `"#binvox 1"`, abort with an error [S2, unofficial].
3.  **Parse Dimensions:** Read the second line. If it does not start with `"dim "`, abort. Parse the three integers following `"dim "`. Let these be $D_x, D_y, D_z$. Set $dim = D_x$ (sources assume cubic $512 \times 512 \times 512$ in examples, but general parsing extracts all three) [S2, S3, unofficial].
4.  **Parse Metadata:**
    - Read the next line as `translate`. Parse three floating-point numbers for $T_x, T_y, T_z$ [S3, unofficial].
    - Read the next line as `scale`. Parse one floating-point number for $S$ [S3, unofficial].
    - Read the next line containing the text `"data"`. This marks the end of the header [S2, S3, unofficial].
5.  **Allocate Storage:**
    - For dense grid: Allocate `mVoxels` of size $D_x \times D_y \times D_z$ [S3, unofficial].
    - For SVO: Allocate flat `data` array of size $dim^3$ initialized to `0` [S2, unofficial].
6.  **Read RLE Data:** Read the remainder of the file into a byte buffer `rle_data` [S2, unofficial].
    - *Pitfall:* Whitespace skipping must be disabled during this phase. The binary data contains valid zero bytes that represent run lengths; skipping whitespace will misalign the RLE pairs [S1, unofficial].
7.  **Decompress RLE:**
    - Iterate through `rle_data` in pairs. For each pair at index $i$ (where $i$ is even):
        - $value = rle\_data[i]$
        - $count = rle\_data[i+1]$
    - If $value \neq 0$:
        - For dense grid: Fill `mVoxels` from current index to $current + count$ with $value$ [S3, unofficial].
        - For SVO: Fill `data` from current index to $current + count$ with $value$. Check if $current + count > dim^3$; if so, abort [S2, unofficial].
    - Update current index by adding $count$ [S2, S3, unofficial].
8.  **Construct Hierarchy (SVO Only):**
    - Initialize a stack with root node $(0, 0, 0, 0, 8)$ representing $(x, y, z, node\_ref, lod)$ [S2, unofficial].
    - While stack is not empty:
        - Pop $(x, y, z, parent\_ref, lod)$ [S2, unofficial].
        - Get parent node. Resize nodes array if necessary [S2, unofficial].
        - Calculate 8 child offsets based on $x, y, z$ and $lod$ [S2, unofficial].
        - If $lod == 0$ (Leaf Level):
            - Initialize `node.occupancy` to `0` [S2, unofficial].
            - For each child $i \in [0, 7]$:
                - Calculate flat index in `data` corresponding to the child's position in the $dim^3$ grid [S2, unofficial].
                - If `data[index] != 0`:
                    - Set bit $i$ in `node.occupancy`.
                    - Set `children[i]` to a new node reference.
                    - Increment `placed_voxels` [S2, unofficial].
            - If `occupancy == 0`, set `node.data.color` to `0` [S2, unofficial].
        - If $lod > 0$ (Internal Node):
            - Set `node.data.color` to `0xff00ffff` [S2, unofficial].
            - Set `node.occupancy` to `0xff` (all children exist) [S2, unofficial].
            - For each child $i \in [0, 7]$:
                - Create new node reference.
                - Push $(offset.x, offset.y, offset.z, child\_ref, lod-1)$ to stack [S2, unofficial].

**Formulas:**
- Linear Index (Dense Grid, S3):
  $$index = x \cdot (Z \cdot Y) + z \cdot Z + y$$
  Where $X, Y, Z$ are dimensions from `mDimensions`. Note: S3 uses $xmult = Z \cdot Y$ and $zmult = Z$.
- Linear Index (SVO Leaf, S2):
  Derived from the flat `data` array populated by RLE. The exact mapping depends on the RLE fill order (typically row-major or similar). S2 implies filling `data` sequentially during RLE decompression.

**Parameters:**
- $dim$: Cubic dimension from header. Sources note it must be equal for X, Y, Z. Exact power-of-2 requirement is noted as a TODO in S1, but examples use 512 [S1, S2, unofficial].
- $xmult$: Integer multiplier for X index, $Z \cdot Y$ [S3, unofficial].
- $zmult$: Integer multiplier for Z index, $Z$ [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Whitespace:** Disabling whitespace skipping is critical for the binary RLE section [S1, unofficial].
- **Dimension Mismatch:** S2 hardcodes panic on non-512 dimensions. General implementation should handle arbitrary $D_x, D_y, D_z$ [S2, S3, unofficial].
- **RLE Bounds:** Ensure $current + count$ does not exceed allocated size [S2, unofficial].
- **Variable Counts:** RLE counts are per-byte-value. The reader must handle variable counts correctly to avoid out-of-bounds reads [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_binvox(file_bytes: bytes) -> dict:
    """
    Parses a binvox file into a dense voxel grid and metadata.
    Returns a dictionary with 'voxels' (bytearray), 'dim', 'translate', 'scale'.
    """
    reader = BufferedBinaryReader(file_bytes)
    
    # 1. Read and validate header
    header_line = reader.read_line()
    if not header_line.startswith("#binvox 1"):
        raise ValueError("Invalid binvox header")
    
    dim_line = reader.read_line()
    if not dim_line.startswith("dim "):
        raise ValueError("Missing dimension line")
    
    parts = dim_line.split()
    if len(parts) != 4:
        raise ValueError("Invalid dimension format")
    
    dim_x = int(parts[1])
    dim_y = int(parts[2])
    dim_z = int(parts[3])
    
    # 2. Read metadata
    translate_line = reader.read_line()
    translate = parse_vector3(translate_line)
    
    scale_line = reader.read_line()
    scale = float(scale_line.split()[1])
    
    data_line = reader.read_line()
    if data_line.strip() != "data":
        raise ValueError("Missing 'data' marker")
    
    # 3. Read RLE data
    # Disable whitespace skipping for binary data
    rle_data = reader.read_all()
    
    # 4. Decompress RLE
    total_voxels = dim_x * dim_y * dim_z
    voxels = bytearray(total_voxels)
    current_idx = 0
    
    i = 0
    while i < len(rle_data) - 1:
        value = rle_data[i]
        count = rle_data[i+1]
        
        if value != 0:
            # Fill voxels with value
            for k in range(count):
                if current_idx + k < total_voxels:
                    voxels[current_idx + k] = value
                else:
                    raise ValueError("RLE data exceeds grid size")
        
        current_idx += count
        i += 2
        
    return {
        'voxels': voxels,
        'dim': (dim_x, dim_y, dim_z),
        'translate': translate,
        'scale': scale
    }

def parse_vector3(line: str) -> tuple:
    parts = line.split()
    return (float(parts[1]), float(parts[2]), float(parts[3]))
```

## Subtopics

### [SVO Construction](link-to-svo-construction.md)
Builds a Sparse Voxel Octree from the dense voxel data. Sets itself apart by using a hierarchical structure for memory efficiency. Choose when memory is constrained or for ray tracing.

### [Schematic Conversion](link-to-schematic-conversion.md)
Converts binvox data into a Schematic object for game engines. Sets itself apart by mapping voxels to color palettes and world coordinates. Choose when exporting to Minecraft-like engines.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels ($dim^3$). RLE decompression is linear in the size of the compressed data, which is $\le N$. SVO construction is $O(N)$ as each voxel is visited once.
- **Memory Complexity:** $O(N)$ for the dense grid. $O(V)$ for SVO where $V$ is the number of occupied voxels (plus overhead for nodes).
- **Measured Performance:** S2 uses 512x512x512 grids. S1 notes power-of-2 requirements as a TODO, implying potential optimization for such sizes.
- **Strengths:** Compact storage via RLE; simple ASCII header for easy debugging.
- **Limitations:** RLE is inefficient for sparse data with many single-voxel changes; fixed header structure limits metadata extensibility.
- **When to Use:** Importing pre-computed voxel models from tools like MagicaVoxel.
- **When Not to Use:** Real-time generation of voxel data; highly sparse large worlds (use SVO directly).

## Background
The binvox format was developed for efficient storage of voxel models, particularly for computer graphics and game development. It is widely used in conjunction with Sparse Voxel Octree (SVO) renderers [S1, S2, unofficial]. The format's simplicity (ASCII header, binary RLE payload) makes it a common interchange format. S2 and S3 demonstrate its use in CPU-based SVO rasterizers and schematic converters, respectively.

## Sources
[S1] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S2] marceline-cramer/svo-cpu, https://github.com/marceline-cramer/svo-cpu, fetched 2026-10-27 (unofficial)
[S3] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
