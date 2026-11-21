# Greedy Meshing with Block Type Masking

Also called: Greedy meshing with multiple block types, Greedy meshing with orientation bits

## Overview
Greedy Meshing with Block Type Masking extends [Greedy Meshing for Voxel Faces](3367-greedy-meshing-for-voxel-faces.md) by ensuring that adjacent voxel faces are merged into a single quad only if they share the same block type (e.g., material, texture) and orientation. Unlike standard greedy meshing, which merges all coplanar faces regardless of material, this method preserves visual distinctions between different block types (e.g., keeping grass and dirt transitions separate) while still maximizing quad size within homogeneous regions.

## How it works
**Inputs:** A voxel grid where each voxel has an associated block type identifier and orientation.
**Outputs:** A set of quads, each associated with a specific block type and orientation.

**Data Structures:**
The core modification is to the mask array used in the greedy meshing algorithm. Instead of a boolean or simple integer indicating presence, the mask stores an integer encoding both block type and orientation.
- **Mask Layout:** An integer array where each element corresponds to a voxel position.
- **Bit Packing:**
  - At least 1 bit is reserved for orientation.
  - The remaining bits store the block type (color, material ID, etc.).
  - *Note:* The exact number of bits for type depends on the maximum number of block types supported.

**Algorithm Steps:**
1. **Initialize Mask:** Create a mask array of the same dimensions as the voxel grid. For each voxel, compute an integer value by combining its orientation bit and block type bits.
2. **Scan for Quads:** Iterate through the voxel grid to find exposed faces.
3. **Group by Type:** When scanning for a greedy quad, compare the mask values of adjacent voxels. Only group voxels into a single quad if their mask values are identical (i.e., same type and orientation).
4. **Mesh Generation:** Perform the greedy meshing process separately for each unique block type/orientation combination. This effectively partitions the voxel surface into homogeneous regions and merges faces within each region.
5. **Output:** Generate quads for each merged region. Each quad retains the block type information for rendering.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_with_block_types(voxel_grid, block_types, orientations):
    """
    voxel_grid: 3D array of booleans indicating voxel presence
    block_types: 3D array of integers representing block type IDs
    orientations: 3D array of integers representing orientation bits
    Returns: List of quads, each with position, size, type, and orientation
    """
    # Step 1: Initialize mask with combined type and orientation
    # Mask value = (orientation << type_bits) | block_type_id
    # Assuming 1 bit for orientation and remaining bits for type
    type_bits = 31  # Example: 31 bits for type, 1 bit for orientation
    mask = zeros_like(voxel_grid, dtype=int)
    for x, y, z in voxel_grid.nonzero():
        mask[x, y, z] = (orientations[x, y, z] << type_bits) | block_types[x, y, z]

    quads = []
    visited = zeros_like(voxel_grid, dtype=bool)

    # Step 2 & 3: Scan and group by type
    for x, y, z in voxel_grid.nonzero():
        if visited[x, y, z]:
            continue
        
        current_mask = mask[x, y, z]
        current_type = current_mask & ((1 << type_bits) - 1)
        current_orient = (current_mask >> type_bits) & 1
        
        # Find maximal quad of same type and orientation
        # This involves scanning in 2D plane perpendicular to face normal
        # For simplicity, assume we are processing a specific face direction
        # In practice, this loop iterates over all 6 face directions
        
        # Example: Processing top face (normal +Y)
        if is_exposed_top(x, y, z, voxel_grid):
            # Find width and height of contiguous region with same mask
            width = 1
            while x + width < grid_width and not visited[x + width, y, z] and \
                  mask[x + width, y, z] == current_mask:
                width += 1
            
            height = 1
            while y + height < grid_height and not visited[x, y + height, z] and \
                  mask[x, y + height, z] == current_mask:
                height += 1
            
            # Create quad
            quad = {
                'position': (x, y, z),
                'size': (width, height),
                'type': current_type,
                'orientation': current_orient,
                'normal': (0, 1, 0)  # Example for top face
            }
            quads.append(quad)
            
            # Mark as visited
            for i in range(width):
                for j in range(height):
                    visited[x + i, y + j, z] = True

    return quads
```

**Parameters:**
- `type_bits`: Number of bits allocated for block type. Must be sufficient to represent all unique block types.
- `orientation_bits`: Number of bits for orientation. Minimum 1.

**Edge Cases and Pitfalls:**
- **Vertex Splitting:** If using a single draw call with a texture atlas, vertices shared by multiple block types must be split. Each copy of the vertex must have distinct texture coordinates pointing to the correct region in the atlas [S1, unofficial].
- **Texture Wrapping:** If using a flat 2D atlas, hardware sampling does not support wrapping for arbitrary atlas regions. The shader must manually handle texture wrapping using shader math [S1, unofficial].
- **Degenerate Input:** Ensure that block type IDs are consistent and that orientation bits are correctly aligned with the face normal being processed.

## Subtopics

### Multi-Material Sub-mesh Rendering
A variant where each group of merged faces of the same block type is rendered as a separate sub-mesh. Each sub-mesh is assigned a distinct material corresponding to its block type and rendered in its own draw call. This approach simplifies texture coordinate management but increases draw call overhead [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the number of voxels. The algorithm scans each voxel a constant number of times (once per face direction).
**Memory Complexity:** $O(N)$ for the mask and visited arrays.
**Performance:** Reduces the number of triangles compared to raw voxel rendering by merging coplanar faces of the same type. However, it may result in more draw calls if using separate sub-meshes per type.
**Strengths:** Preserves visual distinctions between block types while optimizing mesh size. Simple to implement by modifying the mask array.
**Limitations:** Increased complexity in texture coordinate management if using a single draw call. Potential for more draw calls if using sub-meshes.
**When to Use:** When rendering voxel worlds with multiple distinct block types (e.g., grass, dirt, stone) and visual fidelity between types is important.
**When Not to Use:** When performance is critical and block types are not visually distinct, or when the overhead of multiple draw calls is unacceptable.

## Background
This method was discussed in the context of optimizing voxel mesh rendering in Unity, specifically addressing how to handle different block types on an optimized voxel mesh. It builds on the principles of greedy meshing introduced in earlier works on voxel surface extraction.

## Sources
[S1] Texturing different block types on an optimized voxel mesh, https://gamedev.stackexchange.com/questions/185036, fetched 2026-10-30 (unofficial)
