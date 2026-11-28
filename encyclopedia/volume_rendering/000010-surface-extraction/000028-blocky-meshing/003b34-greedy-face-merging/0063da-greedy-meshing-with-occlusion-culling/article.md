# Greedy Meshing with Occlusion Culling

Also called: Face Mask Culling Meshing, VoxelEngine Greedy Mesher

## Overview
This method converts voxel data into optimized quads by grouping contiguous voxels of the same material on each face, while using bitwise occlusion masks to cull faces hidden by other geometry [S1, unofficial]. It differs from standard greedy meshing by pre-computing face-specific occlusion masks using bitwise operations on a global opaque mask, rather than checking neighbors during the meshing sweep [S1, unofficial]. This approach is particularly effective for chunk-based voxel rendering where interior faces are frequently occluded [S1, unofficial].

## How it works
The algorithm operates on a voxel chunk of fixed dimensions, typically $32 \times 32 \times 32$ [S1, unofficial]. It processes each of the six face normals independently, generating bitmasks that represent visible surfaces after occlusion culling, then applies greedy expansion on these masks.

### Inputs and Outputs
*   **Input:** A voxel chunk of size $CHUNK\_SIZE \times CHUNK\_SIZE \times CHUNK\_SIZE$ (typically 32). Each voxel has a material ID and an opacity flag (opaque/transparent).
*   **Output:** A list of quads. Each quad is defined by a 3D start position (chunk-local), a material ID, and two length dimensions.

### Data Structures
The core data structure for visibility is a 3D array of bitmasks, `faceMasks`.
*   **Layout:** `faceMasks[8][CHUNK_SIZE][CHUNK_SIZE]` [S1, unofficial].
    *   Indices 0-1: Y-axis faces (Normal -Y, +Y).
    *   Indices 2-3: Z-axis faces (Normal -Z, +Z).
    *   Indices 4-5: X-axis faces (Normal -X, +X). These are transposed relative to the voxel grid for cache locality.
    *   Indices 6-7: Intermediate X-axis masks used for transposition.
*   **Mask Type:** `Bitmap<CHUNK_SIZE>`, which is a 32-bit integer (or equivalent fixed-width integer) where each bit represents a voxel along one axis [S1, unofficial].
*   **Quad Record:** A packed 32-bit integer structure `Quad` [S1, unofficial]:
    *   Bits 31-27 (5 bits): Start X position.
    *   Bits 26-22 (5 bits): Start Y position.
    *   Bits 21-17 (5 bits): Start Z position.
    *   Bits 16-10 (7 bits): Material ID.
    *   Bits 9-5 (5 bits): Length X.
    *   Bits 4-0 (5 bits): Length Y (or Z, depending on face normal).

### Steps

1.  **Compute Global Opaque Mask:** Create a 3D bitfield `opaqueMask` where bit $i$ is 1 if voxel $i$ is opaque, 0 otherwise.
2.  **Generate Face Masks:** For each face normal, compute a 2D mask of visible faces by comparing adjacent slices in the opaque mask.
    *   **Y-Axis Faces (0, 1):**
        *   For face 0 (-Y): If $y > 0$, `faceMask[0][y][z] = opaqueMask[y][z] & (~opaqueMask[y-1][z])`. Else, `faceMask[0][y][z] = opaqueMask[y][z]`.
        *   For face 1 (+Y): If $y < CHUNK\_SIZE-1$, `faceMask[1][y][z] = opaqueMask[y][z] & (~opaqueMask[y+1][z])`. Else, `faceMask[1][y][z] = opaqueMask[y][z]`.
    *   **Z-Axis Faces (2, 3):**
        *   For face 2 (-Z): If $z > 0$, `faceMask[2][z][y] = opaqueMask[y][z] & (~opaqueMask[y][z-1])`. Else, `faceMask[2][z][y] = opaqueMask[y][z]`. Note the index swap $[z][y]$ for cache locality.
        *   For face 3 (+Z): If $z < CHUNK\_SIZE-1$, `faceMask[3][z][y] = opaqueMask[y][z] & (~opaqueMask[y][z+1])`. Else, `faceMask[3][z][y] = opaqueMask[y][z]`.
    *   **X-Axis Intermediate Masks (6, 7):**
        *   `faceMask[6][y][z] = opaqueMask[y][z] & ~(opaqueMask[y][z] << 1)` (Left edge).
        *   `faceMask[7][y][z] = opaqueMask[y][z] & ~(opaqueMask[y][z] >> 1)` (Right edge).
3.  **Transpose X-Axis Masks:** To process X-axis faces uniformly with the 2D mask logic, transpose the intermediate masks into `faceMask[4]` and `faceMask[5]`.
    *   For each $y, x$:
        *   `masks[0] = 0`, `masks[1] = 0`.
        *   For each $z$ from 0 to $CHUNK\_SIZE-1$:
            *   `bit6 = (faceMask[6][y][z] >> x) & 1`
            *   `bit7 = (faceMask[7][y][z] >> x) & 1`
            *   `masks[0] |= (bit6 << z)`
            *   `masks[1] |= (bit7 << z)`
        *   `faceMask[4][y][x] = masks[0]`
        *   `faceMask[5][y][x] = masks[1]`
4.  **Greedy Expansion per Face:** For each of the 6 face masks, perform greedy meshing.
    *   Iterate through the 2D mask (rows and columns).
    *   Find the first non-zero bit in the current row/column using `trailing_zeroes` or equivalent.
    *   **Pitfall:** If `trailing_zeroes` returns $CHUNK\_SIZE$, the entire row/column is empty; skip to the next index.
    *   Identify the start position and material ID of the voxel corresponding to the first set bit.
    *   **Extend Length:** Move in the primary direction (e.g., X for Y/Z faces, Z for X faces) while:
        *   The corresponding bit in the mask is set.
        *   The material ID of the voxel matches the start material.
    *   **Extend Width:** Move in the secondary direction while all bits in the rectangle defined by the current start, length, and width are set and match the material ID.
    *   **Commit Quad:** Create a `Quad` record with start position, material, and lengths.
    *   **Clear Mask:** Clear the bits corresponding to the committed quad from the `faceMask` to prevent double-counting.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_meshing_with_occlusion_culling(volume, dims, opaque_mask):
    """
    volume: function (x, y, z) -> material_id
    dims: tuple (Dx, Dy, Dz), typically (32, 32, 32)
    opaque_mask: 3D array of booleans or bitmasks indicating opaque voxels
    """
    CHUNK_SIZE = dims[0]
    quads = []
    
    # faceMasks[face][y][z] or [y][x] for X-axis
    # 0:-Y, 1:+Y, 2:-Z, 3:+Z, 4:-X, 5:+X, 6:intermediate-X-left, 7:intermediate-X-right
    face_masks = [[[0] * CHUNK_SIZE for _ in range(CHUNK_SIZE)] for _ in range(8)]
    
    # Step 2: Generate Face Masks
    for y in range(CHUNK_SIZE):
        for z in range(CHUNK_SIZE):
            # Y-axis faces
            if y > 0:
                face_masks[0][y][z] = opaque_mask[y][z] & ~opaque_mask[y-1][z]
            else:
                face_masks[0][y][z] = opaque_mask[y][z]
            
            if y < CHUNK_SIZE - 1:
                face_masks[1][y][z] = opaque_mask[y][z] & ~opaque_mask[y+1][z]
            else:
                face_masks[1][y][z] = opaque_mask[y][z]
                
            # Z-axis faces (indices swapped for cache locality)
            if z > 0:
                face_masks[2][z][y] = opaque_mask[y][z] & ~opaque_mask[y][z-1]
            else:
                face_masks[2][z][y] = opaque_mask[y][z]
                
            if z < CHUNK_SIZE - 1:
                face_masks[3][z][y] = opaque_mask[y][z] & ~opaque_mask[y][z+1]
            else:
                face_masks[3][z][y] = opaque_mask[y][z]
                
            # X-axis intermediate masks
            # Assuming opaque_mask[y][z] is a bitmask where bit x corresponds to voxel x
            # This requires opaque_mask to be pre-computed as bitmasks per y,z slice
            face_masks[6][y][z] = opaque_mask[y][z] & ~(opaque_mask[y][z] << 1)
            face_masks[7][y][z] = opaque_mask[y][z] & ~(opaque_mask[y][z] >> 1)

    # Step 3: Transpose X-Axis Masks
    for y in range(CHUNK_SIZE):
        for x in range(CHUNK_SIZE):
            mask0 = 0
            mask1 = 0
            for z in range(CHUNK_SIZE):
                bit6 = (face_masks[6][y][z] >> x) & 1
                bit7 = (face_masks[7][y][z] >> x) & 1
                mask0 |= (bit6 << z)
                mask1 |= (bit7 << z)
            face_masks[4][y][x] = mask0
            face_masks[5][y][x] = mask1

    # Step 4: Greedy Expansion
    # Process Y and Z faces (0-3)
    for face in range(4):
        for row in range(CHUNK_SIZE):
            for col in range(CHUNK_SIZE):
                mask = face_masks[face][row][col]
                if mask == 0:
                    continue
                
                # Find first set bit
                bit_pos = mask & -mask  # Isolate lowest set bit
                if bit_pos == 0:
                    continue
                    
                # Determine start position and material based on face and bit_pos
                # This part depends on the specific mapping of bit_pos to 3D coordinates
                # For simplicity, assume we can derive start_x, start_y, start_z and material
                start_pos, material = get_start_from_mask(face, row, col, bit_pos, volume)
                
                # Extend length (primary direction)
                length = 1
                while length < CHUNK_SIZE:
                    next_pos = get_next_pos_in_direction(face, start_pos, length)
                    if not is_opaque(next_pos, opaque_mask) or volume(*next_pos) != material:
                        break
                    length += 1
                
                # Extend width (secondary direction)
                width = 1
                while width < CHUNK_SIZE:
                    # Check if entire row/col segment of length 'length' is valid
                    valid = True
                    for l in range(length):
                        check_pos = get_pos_offset(start_pos, face, l, width)
                        if not is_bit_set_in_mask(face_masks[face], check_pos) or volume(*check_pos) != material:
                            valid = False
                            break
                    if not valid:
                        break
                    width += 1
                
                # Commit Quad
                quads.append(Quad(start_pos, material, length, width))
                
                # Clear processed bits
                clear_bits_in_mask(face_masks[face], start_pos, length, width)

    # Process X faces (4-5) - similar logic but with transposed coordinates
    for face in range(4, 6):
        for row in range(CHUNK_SIZE):
            for col in range(CHUNK_SIZE):
                mask = face_masks[face][row][col]
                if mask == 0:
                    continue
                
                bit_pos = mask & -mask
                if bit_pos == 0:
                    continue
                    
                start_pos, material = get_start_from_mask(face, row, col, bit_pos, volume)
                
                # Extend length (Z direction for X faces)
                length = 1
                while length < CHUNK_SIZE:
                    next_pos = get_next_pos_in_direction(face, start_pos, length)
                    if not is_opaque(next_pos, opaque_mask) or volume(*next_pos) != material:
                        break
                    length += 1
                
                # Extend width (Y direction for X faces)
                width = 1
                while width < CHUNK_SIZE:
                    valid = True
                    for l in range(length):
                        check_pos = get_pos_offset(start_pos, face, l, width)
                        if not is_bit_set_in_mask(face_masks[face], check_pos) or volume(*check_pos) != material:
                            valid = False
                            break
                    if not valid:
                        break
                    width += 1
                
                quads.append(Quad(start_pos, material, length, width))
                clear_bits_in_mask(face_masks[face], start_pos, length, width)

    return quads

def get_start_from_mask(face, row, col, bit_pos, volume):
    # Helper to map mask coordinates to 3D voxel coordinates and material
    # Implementation details depend on specific bit-to-voxel mapping
    # For face 0 (-Y): row=y, col=z, bit_pos=x
    # For face 2 (-Z): row=z, col=y, bit_pos=x
    # For face 4 (-X): row=y, col=x, bit_pos=z (transposed)
    pass

def is_bit_set_in_mask(face_mask, pos):
    # Helper to check if a specific voxel's face bit is set in the mask
    pass

def clear_bits_in_mask(face_mask, start_pos, length, width):
    # Helper to clear bits in the mask for the committed quad
    pass
```

## Operating characteristics
*   **Time Complexity:** $O(V)$ where $V$ is the number of voxels, as each voxel is processed a constant number of times (once for mask generation, once for greedy expansion). The bitwise operations are efficient, typically $O(1)$ per word.
*   **Memory Complexity:** $O(CHUNK\_SIZE^2)$ for the face masks, plus $O(Q)$ for the output quads, where $Q$ is the number of quads.
*   **Performance:** The bitwise occlusion culling significantly reduces the number of quads generated for large, solid chunks compared to standard greedy meshing, which checks neighbors for every voxel. This leads to fewer draw calls and better GPU utilization [S1, unofficial].
*   **Strengths:** Efficient occlusion culling, compact quad representation, suitable for chunk-based rendering.
*   **Limitations:** Requires pre-computation of opaque masks, which adds overhead. The transposition step for X-axis faces adds complexity. Not suitable for dynamic voxel changes without recomputing masks.

## Background
This method is derived from the VoxelEngine project, which implements a voxel engine using modern OpenGL features like instancing and indirect drawing [S1, unofficial]. The use of bitwise masks for occlusion culling is a common optimization in voxel rendering to reduce overdraw and draw calls. The specific implementation details, including the quad packing and mask transposition, are tailored for performance on modern GPUs.

## Sources
[S1] IVSOP/VoxelEngine, https://github.com/IVSOP/VoxelEngine, fetched 2026-10-26 (unofficial)
