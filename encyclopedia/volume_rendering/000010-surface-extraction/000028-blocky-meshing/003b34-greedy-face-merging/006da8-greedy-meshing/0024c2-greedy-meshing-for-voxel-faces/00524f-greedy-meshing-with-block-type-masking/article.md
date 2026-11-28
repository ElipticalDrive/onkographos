# Greedy Meshing with Block Type Masking

Also called: Greedy meshing with multiple block types, Greedy meshing with orientation bits

## Overview
Greedy Meshing with Block Type Masking modifies [Greedy Meshing for Voxel Faces](3367-greedy-meshing-for-voxel-faces.md) by grouping adjacent voxels into single quads only if they share the same block type and orientation [S1, unofficial]. Unlike standard greedy meshing, which merges all coplanar faces regardless of material, this method preserves visual distinctions between different block types (e.g., keeping grass and dirt transitions separate) while still maximizing quad size within homogeneous regions [S1, unofficial].

## How it works
**Inputs:** A voxel grid where each voxel has an associated block type identifier and orientation [G].
**Outputs:** A set of quads, each associated with a specific block type and orientation [G].

**Data Structures:**
The core modification is to the mask array used in the greedy meshing algorithm [S1, unofficial]. Instead of a boolean or simple integer indicating presence, the mask stores an integer encoding both block type and orientation [S1, unofficial].
- **Mask Layout:** An integer array where each element corresponds to a voxel position [S1, unofficial].
- **Bit Packing:** At least 1 bit is reserved for orientation, and the remaining bits store the block color or type [S1, unofficial]. The exact number of bits for type depends on the maximum number of block types supported [G].

**Formulas:**
- Mask encoding: `mask_val = (orientation << type_bits) | block_type_id`
- Type extraction: `block_type_id = mask_val & ((1 << type_bits) - 1)`
- Orientation extraction: `orientation = (mask_val >> type_bits) & 1`

**Algorithm Steps:**
1. **Initialize Mask:** Create a mask array of the same dimensions as the voxel grid. For each voxel, compute an integer value by combining its orientation bit and block type bits [S1, unofficial].
2. **Scan for Quads:** Iterate through the voxel grid to find exposed faces [G].
3. **Group by Type:** When scanning for a greedy quad, compare the mask values of adjacent voxels. Only group voxels into a single quad if their mask values are identical (i.e., same type and orientation) [S1, unofficial].
4. **Mesh Generation:** Perform the greedy meshing process separately for each unique block type/orientation combination. This effectively partitions the voxel surface into homogeneous regions and merges faces within each region [S1, unofficial].
5. **Output:** Generate quads for each merged region. Each quad retains the block type information for rendering [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_block_types(grid, W, H, D, type_bits):
    """
    grid: 3D list of integers where each integer is a packed mask (orientation << type_bits) | block_type_id
          0 represents an empty voxel.
    W, H, D: grid dimensions along X, Y, Z axes.
    type_bits: number of bits allocated for block type.
    Returns: List of quad dictionaries with 'position', 'size', 'mask', and 'normal'.
    """
    visited = [[[False] * D for _ in range(H)] for _ in range(W)]
    quads = []
    
    # The full algorithm iterates over all 6 face normals.
    # This example implements the +Y (top) face direction.
    for x in range(W):
        for y in range(H):
            for z in range(D):
                if visited[x][y][z]:
                    continue
                # Check if exposed on the +Y side
                if y + 1 >= H or grid[x][y + 1][z] == 0:
                    current_mask = grid[x][y][z]
                    
                    # Expand along X axis
                    width = 0
                    while x + width < W and not visited[x + width][y][z] and grid[x + width][y][z] == current_mask:
                        width += 1
                    
                    # Expand along Z axis
                    height = 0
                    while z + height < D and not visited[x][y][z + height] and grid[x][y][z + height] == current_mask:
                        height += 1
                    
                    # Record quad
                    quads.append({
                        'position': (x, y, z),
                        'size': (width, height),
                        'mask': current_mask,
                        'normal': (0, 1, 0)
                    })
                    
                    # Mark cells as visited
                    for i in range(width):
                        for j in range(height):
                            visited[x + i][y][z + j] = True
    return quads
```

**Parameters:**
- `type_bits`: Number of bits allocated for block type. Must be sufficient to represent all unique block types [G].
- `orientation_bits`: Number of bits for orientation. Minimum 1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Vertex Splitting:** If using a single draw call with a texture atlas, vertices shared by multiple block types must be split. Each copy of the vertex must have distinct texture coordinates pointing to the correct region in the atlas [S1, unofficial].
- **Texture Wrapping:** If using a flat 2D atlas, hardware sampling does not support wrapping for arbitrary atlas regions. The shader must manually handle texture wrapping using shader math [S1, unofficial].
- **Degenerate Input:** Ensure that block type IDs are consistent and that orientation bits are correctly aligned with the face normal being processed [G].

## Subtopics

### Multi-Material Sub-mesh Rendering
A variant where each group of merged faces of the same block type is rendered as a separate sub-mesh. Each sub-mesh is assigned a distinct material corresponding to its block type and rendered in its own draw call [S1, unofficial]. This approach increases draw call overhead but avoids texture coordinate splitting and atlas wrapping complexity [G].

## Operating characteristics
**Time Complexity:** Not given in the sources. Standard greedy meshing runs in $O(N)$ time where $N$ is the number of voxels, as each voxel is visited a constant number of times [G].
**Memory Complexity:** Not given in the sources. Requires $O(N)$ space for the mask and visited arrays [G].
**Performance:** Reduces the number of triangles compared to raw voxel rendering by merging coplanar faces of the same type [S1, unofficial]. May result in more draw calls if using separate sub-meshes per type [S1, unofficial].
**Strengths:** Preserves visual distinctions between block types while optimizing mesh size. Simple to implement by modifying the mask array [S1, unofficial].
**Limitations:** Increased complexity in texture coordinate management if using a single draw call. Potential for more draw calls if using sub-meshes [S1, unofficial].
**When to Use:** When rendering voxel worlds with multiple distinct block types (e.g., grass, dirt, stone) and visual fidelity between types is important [G].
**When Not to Use:** When performance is critical and block types are not visually distinct, or when the overhead of multiple draw calls is unacceptable [G].

## Background
This method was discussed in the context of optimizing voxel mesh rendering in Unity, specifically addressing how to handle different block types on an optimized voxel mesh [S1, unofficial]. It builds on the principles of greedy meshing introduced in earlier works on voxel surface extraction [G].

## Sources
[S1] Texturing different block types on an optimized voxel mesh, https://gamedev.stackexchange.com/questions/185036, fetched 2026-10-30 (unofficial)
