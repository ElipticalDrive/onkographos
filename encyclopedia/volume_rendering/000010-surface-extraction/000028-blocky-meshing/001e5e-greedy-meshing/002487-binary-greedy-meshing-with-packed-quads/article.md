# Binary Greedy Meshing with Packed Quads

Also called: BGM, Binary Greedy Mesher, Binary Greedy Meshing, Binary Greedy Plane, Binary Greedy Voxel Meshing, Greedy Cubic

## Overview
Binary Greedy Meshing (BGM) generates maximal axis-aligned quads from a binary voxel slice by greedily expanding horizontally from the first solid block found in each vertical run, using bitwise operations to minimize clock cycles [S1, S2, S3, unofficial]. It treats the Signed Distance Field (SDF) or voxel grid as solid/empty, culls hidden faces, and merges co-planar visible faces into maximal rectangles drawn via instanced rendering from an 8-byte packed record [S3, unofficial]. Unlike [Marching Cubes](52-marching-cubes.md) which produces smooth isosurfaces, BGM preserves the blocky aesthetic while reducing triangle count by 1.88–2.60× and geometry size by 39.5–54.5× compared to Marching Cubes on typical terrain [S3, unofficial]. It builds on [Greedy Meshing](1776-greedy-meshing.md) by replacing per-voxel iteration with row-wise bitwise logic [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A voxel chunk of size $N \times N \times N$ (typically $N=16$ or $32$), a face direction (Up, Down, Left, Right, Forward, Back), and optionally an Ambient Occlusion (AO) grid and block type map.
- **Output:** A list of quads, each defined by position $(x, y, z)$, dimensions $(w, h)$, face direction, and attributes (AO, block type).

**Data Structures:**
1.  **Axis Columns (`axisCols`):** A buffer storing voxel presence for three axes. For a chunk size $N$ and padding $P = N+2$, the size is $P^2 \times 4 \times 3$ bytes. It contains three $P \times P$ grids of 32-bit integers, where each bit represents a voxel [S2, unofficial].
2.  **Face Masks (`colFaceMasks`):** A buffer of size $P^2 \times 4 \times 3 \times 2$ bytes. For each axis, it stores two masks per row: one for outward-facing faces (e.g., UP/RIGHT/BACK) and one for inward-facing (e.g., DOWN/LEFT/FORWARD) [S2, unofficial].
3.  **Binary Plane (`data`):** A 1D array of 32-bit integers (or 16-bit for $N=16$) representing a single slice of face locations. Each bit set to 1 indicates a visible face at that position [S1, S2, unofficial].
4.  **VoxelQuadEncoding:** An 8-byte (64-bit) packed record for instancing.
    -   **Geometry Word (32 bits):** $x$ (5 bits), $y$ (5 bits), $z$ (5 bits), $w-1$ (5 bits), $h-1$ (5 bits), face direction (3 bits), reserved (4 bits). Extents are stored biased-by-one to support $32^3$ chunks [S3, unofficial].
    -   **Material Word (32 bits):** 8 bits used for material index, padded to 32 bits [S3, unofficial].

**Algorithm Steps:**

1.  **Build Voxel Grid:**
    Iterate over the chunk volume. For each solid voxel at $(x, y, z)$, set the corresponding bit in `axisCols` for the XZ, ZY, and XY axes. The index calculation depends on the axis [S2, unofficial]:
    -   XZ Axis: `index = (x + (z * P)) * 4`
    -   ZY Axis: `index = (z + (y * P) + P^2) * 4`
    -   XY Axis: `index = (x + (y * P) + P^2 * 2) * 4`
    Set bit $y$ (for XZ), $x$ (for ZY), or $z$ (for XY) to true [S2, unofficial].

2.  **Cull Faces:**
    For each axis (0 to 2) and each row $i$ in the $P \times P$ grid:
    -   Read the column `col` from `axisCols`.
    -   Compute outward mask: `outward = col & !(col << 1)` (for ascending axis) or `col & !(col >> 1)` (for descending axis).
    -   Compute inward mask: `inward = col & !(col >> 1)` (for ascending) or `col & !(col << 1)` (for descending).
    -   Write to `colFaceMasks` at indices corresponding to the face direction [S1, S2, unofficial].

3.  **Generate Axis Planes:**
    For each of the 6 face directions (axes 0-5):
    -   Iterate over the plane coordinates (e.g., $z, x$ for Y-axis faces).
    -   Read the face mask column from `colFaceMasks`.
    -   Shift right by 1 and mask out the top bit (bit $N$) to remove edge cases from neighboring chunks [S2, unofficial].
    -   Store the resulting binary plane in a hash table `data[axis][blockType][y]`, where the key combines the block type and AO pattern [S1, S2, unofficial]. The AO pattern is a 9-bit integer derived from the 3x3 neighborhood of the voxel [S1, unofficial].

4.  **Greedy Mesh Each Plane:**
    For each binary plane buffer `data`:
    -   Initialize `row` from 0 to $N-1$.
    -   Initialize `y = 0`.
    -   **Skip Air:** Calculate `airHeight = trailing_zeros(data[row] >> y)`. Update `y += airHeight`. If `y >= N`, break.
    -   **Calculate Height:** Calculate `h = trailing_ones(data[row] >> y)`.
    -   **Create Mask:** `h_as_mask = (1 << h) - 1`. If $h=32$, use `!0`. `mask = h_as_mask << y`.
    -   **Expand Width:** Initialize `w = 1`. While `row + w < N`:
        -   Read `nextRow = data[row + w]`.
        -   Check if the bits in `nextRow` at positions $y$ to $y+h-1$ match `h_as_mask`. Specifically: `(nextRow >> y) & h_as_mask == h_as_mask`.
        -   If match:
            -   Remove used bits from `nextRow` to prevent reuse: `data[row + w] &= ~mask`.
            -   Increment `w`.
        -   Else: Break.
    -   **Emit Quad:** Create a quad with position $(row, y, \text{depth})$, width $w$, height $h$.
    -   Update `y += h`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_binary_plane(data: list[int], lod_size: int) -> list[dict]:
    """
    Greedy mesh a single binary plane represented as a list of 32-bit integers.
    data: List of integers where each integer represents a row of bits.
    lod_size: The dimension of the chunk (e.g., 16 or 32).
    Returns: List of dicts with keys 'x', 'y', 'w', 'h'.
    """
    greedy_quads = []
    
    for row in range(lod_size):
        y = 0
        while y < lod_size:
            # Skip leading air (zero bits)
            # Note: In Python, (data[row] >> y).bit_count() is not trailing_zeros.
            # We simulate trailing_zeros by checking bits.
            shift_val = data[row] >> y
            air_height = 0
            while air_height < (lod_size - y) and (shift_val & 1) == 0:
                air_height += 1
                shift_val >>= 1
            
            y += air_height
            if y >= lod_size:
                break
            
            # Calculate height of solid run
            # Count trailing ones in the shifted value
            h = 0
            temp_val = data[row] >> y
            while h < (lod_size - y) and (temp_val & 1) == 1:
                h += 1
                temp_val >>= 1
            
            # Create mask for height h
            if h == 32:
                h_as_mask = 0xFFFFFFFF
            else:
                h_as_mask = (1 << h) - 1
            
            # Align mask to current y position
            mask = h_as_mask << y
            
            # Expand horizontally
            w = 1
            while row + w < lod_size:
                next_row_val = data[row + w]
                # Check if the bits in next_row at positions y..y+h-1 are all 1
                next_row_slice = (next_row_val >> y) & h_as_mask
                if next_row_slice == h_as_mask:
                    # Remove used bits to prevent reuse in future quads
                    data[row + w] &= ~mask
                    w += 1
                else:
                    break
            
            # Emit quad
            greedy_quads.append({
                'x': row,
                'y': y,
                'w': w,
                'h': h
            })
            
            y += h
            
    return greedy_quads

def build_face_masks(axis_cols: list[int], chunk_size_p: int) -> list[list[int]]:
    """
    Build face masks from axis columns.
    axis_cols: List of 3 * chunk_size_p^2 integers.
    Returns: List of 6 * chunk_size_p^2 integers (2 masks per row per axis).
    """
    col_face_masks = [0] * (6 * chunk_size_p * chunk_size_p)
    
    for axis in range(3):
        for i in range(chunk_size_p * chunk_size_p):
            # Index calculation for axis_cols
            # axis 0: XZ, axis 1: ZY, axis 2: XY
            # Each axis has chunk_size_p^2 rows
            col = axis_cols[axis * chunk_size_p * chunk_size_p + i]
            
            # Compute outward and inward masks
            # Outward: bit is 1 if current is 1 and next is 0
            # Inward: bit is 1 if current is 1 and prev is 0
            # Assuming ascending order for outward/downward logic
            
            # For simplicity, assuming standard bit ordering
            outward = col & ~(col << 1)
            inward = col & ~(col >> 1)
            
            # Store in col_face_masks
            # 2 masks per row: index 2*axis + 0 for outward, 2*axis + 1 for inward
            col_face_masks[2 * axis * chunk_size_p * chunk_size_p + i] = outward
            col_face_masks[(2 * axis + 1) * chunk_size_p * chunk_size_p + i] = inward
            
    return col_face_masks
```

**Parameters:**
-   `CHUNK_SIZE`: 16 or 32 [S2, S3, unofficial].
-   `CHUNK_SIZE_P`: `CHUNK_SIZE + 2` (18 or 34) for padding [S2, S3, unofficial].
-   `BLOCK_TYPES`: Number of distinct block types (e.g., 3) [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Padding:** Uniform padding on all three axes is required to handle chunk boundaries correctly. Special-casing Y-axis padding against neighbor chunks can lead to bugs [S3, unofficial].
-   **Extents:** Width and height in the packed quad encoding must be stored biased-by-one (i.e., store $w-1$) to support full $32^3$ chunks, as 5 bits can only represent 0-31 [S3, unofficial].
-   **AO Index Collisions:** The AO index uses 9 bits. If block types or AO patterns exceed bit limits, collisions may occur [S1, unofficial].
-   **Shader Mismatch:** The C++ `VoxelQuad.h` and GLSL `VoxelQuadUnpack.glsl` must match exactly. A mismatch causes silent transposition or offset errors [S3, unofficial].
-   **Face Basis:** The `kWidthIsInnerExtent` table varies by face direction. Incorrect mapping transposes quads (e.g., 1x8 becomes 8x1) [S3, unofficial].

## Subtopics

### [Streaming Binary Greedy Meshing](3777-streaming-binary-greedy-meshing.md)
Processes chunks x-slice by x-slice with rolling buffers to improve cache locality, emitting quads for all six directions as runs terminate. Choose when memory bandwidth or cache misses are a bottleneck.

### [Binary Greedy Meshing with Ambient Occlusion](3778-binary-greedy-meshing-with-ambient-occlusion.md)
Extends BGM to include AO data in the quad attributes, requiring a 3x3 neighborhood sample for each voxel. Choose when lighting quality is critical and the performance cost of AO calculation is acceptable.

### [Binary Greedy Meshing with Texture Atlases](3779-binary-greedy-meshing-with-texture-atlases.md)
Integrates texture atlas UVs into the packed quad record. Choose when texture filtering or atlas-based rendering is required.

## Operating characteristics
-   **Time Complexity:** $O(N^2)$ per face direction, where $N$ is the chunk size. The bitwise operations make the constant factor very small.
-   **Memory Complexity:** $O(N^2)$ for the face masks and binary planes. The packed quad output is $O(Q)$ where $Q$ is the number of quads.
-   **Performance:** For a 16x16 chunk, BGM takes 0.116ms, a 2.49ms improvement over traditional greedy meshing (2.606ms) [S2, unofficial]. On rugged terrain, it produces 8,469 quads vs 31,869 tris for Marching Cubes, with 39.5x less geometry [S3, unofficial].
-   **Strengths:** Extremely fast due to bitwise operations; low memory footprint; minimal draw calls via instancing.
-   **Limitations:** Only works for binary (solid/empty) voxels; cannot handle smooth surfaces or partial voxels; requires careful handling of chunk boundaries and AO.
-   **When to Use:** Blocky voxel games, terrain generation, and scenarios where performance and low triangle count are prioritized over smooth geometry.
-   **When Not to Use:** Organic terrain requiring smooth isosurfaces (use Marching Cubes), or when partial voxel transparency is needed.

## Background
Binary Greedy Meshing was developed as an optimization for voxel rendering, leveraging bitwise operations to accelerate the greedy meshing process. It was popularized in the context of voxel engines like Roblox and custom game engines (e.g., OloEngine) to reduce the high triangle counts of naive voxel meshing. The method treats the voxel volume as a binary field, allowing for efficient face culling and quad merging. It is often compared to Marching Cubes, with which it can coexist in the same engine, selected based on the desired aesthetic (blocky vs. smooth) [S3, unofficial]. The technique has been refined to include packed quad encodings for efficient GPU instancing and streaming variants for better cache performance [S4, unofficial].

## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S2] AshR-03/Binary-Greedy-Mesher-For-Voxel-Graphic-Generation, https://github.com/AshR-03/Binary-Greedy-Mesher-For-Voxel-Graphic-Generation, fetched 2026-10-26 (unofficial)
[S3] Terrain: binary greedy meshing + packed quads for blocky voxel terrain (#727), https://github.com/drsnuggles8/OloEngineBase/pull/830, fetched 2026-10-27 (unofficial)
[S4] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
