# Voxel Body Copying with Sub-Voxel Clipping

Also called: Clipped Block Copying, VoxelBody.Cropped

## Overview
Voxel Body Copying with Sub-Voxel Clipping extracts a subset of a voxel grid by retaining only the active (solid) blocks that intersect a specified world-space bounding box, while preserving the internal structure of those blocks. Unlike [Grid Clone](7232-grid-clone.md), which duplicates the entire grid, this method creates a new, potentially smaller `VoxelBody` containing only the relevant data. It differs from [Bounding Box Clipping](4643-bounding-box-clipping.md) in that it operates on discrete solid blocks rather than individual voxels, clipping blocks that partially overlap the boundary to maintain structural integrity.

## How it works
**Inputs:**
- Source `VoxelBody` (or equivalent volume data structure).
- World-space bounding box defined by minimum and maximum integer coordinates $(min_x, min_y, min_z)$ and $(max_x, max_y, max_z)$.

**Outputs:**
- A new `VoxelBody` containing only the clipped solid blocks, or `null` if no solid blocks remain.

**Data Structures:**
- `VoxelBody`: A container for a list of `Voxel` objects.
- `Voxel`: Represents a solid block of voxels. Contains a field (e.g., a 3D array or sparse representation) defining the material/opacity within the block.
- `kept`: A list of `Voxel` objects to be included in the output.

**Algorithm:**
1. **Calculate Range:** Determine the integer bounding box of the source volume's solid content using a range query (e.g., `TryGetRange`). If the volume has no solid range, return `null` [S1, unofficial].
2. **Iterate and Clip:** Iterate over all solid blocks in the source volume (e.g., using `ForEachSolidIn`) [S1, unofficial]. For each block:
   - Check if the block is fully inside the target bounding box.
   - If not fully inside, call a clipping function `ClipBlock` with the parameter `keepInside: true`. This function returns a new field representing the portion of the block that lies inside the bounding box [S1, unofficial].
   - If the resulting clipped field is empty, skip the block.
   - If the resulting clipped field is full (identical to the original block), set the field to `null` (or mark as full) to optimize storage [S1, unofficial].
   - Otherwise, create a new `VoxelBody.Voxel` with the clipped field and add it to the `kept` list [S1, unofficial].
3. **Construct Result:** If the `kept` list is empty, return `null`. Otherwise, create a new `VoxelBody` using the constructor `VoxelBody.Cropped`, passing the original name and the `kept` list [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def VoxelBody_CopyWithSubVoxelClipping(source_body, target_bbox_min, target_bbox_max):
    # Step 1: Calculate range
    if not source_body.TryGetRange(out solid_min, out solid_max):
        return None

    kept_voxels = []

    # Step 2: Iterate and clip
    for block in source_body.ForEachSolidIn():
        # Check if block is fully inside target bbox
        if block.is_fully_inside(target_bbox_min, target_bbox_max):
            # Optimization: If fully inside, we can keep the reference or copy
            # The source notes imply we only clip if NOT fully inside, 
            # but for consistency with the "clipped field" logic, we treat it as a special case.
            # However, the note says: "if it is not fully inside, call ClipBlock"
            # So if it IS fully inside, we just add it as is? 
            # The note says: "If it is full, set field to null." This likely refers to the result of ClipBlock.
            # Let's follow the note strictly:
            pass 
        else:
            # Call ClipBlock with keepInside: true
            clipped_field = ClipBlock(block.field, target_bbox_min, target_bbox_max, keepInside=True)
            
            if clipped_field is None or clipped_field.is_empty():
                continue
            
            if clipped_field.is_full():
                voxel = VoxelBody.Voxel(name=block.name, field=None) # Null field implies full
            else:
                voxel = VoxelBody.Voxel(name=block.name, field=clipped_field)
            
            kept_voxels.append(voxel)

    # Step 3: Construct result
    if len(kept_voxels) == 0:
        return None
    
    return VoxelBody.Cropped(name=source_body.name, voxels=kept_voxels)
```

**Parameters:**
- `keepInside`: Boolean. When `true`, retains the part of the block inside the bounding box. When `false`, retains the part outside (used in Carving).

**Edge Cases:**
- **No Solid Range:** If the source volume is empty, the method returns `null` immediately [S1, unofficial].
- **Empty Clipped Field:** If a block is completely outside the bounding box, `ClipBlock` returns an empty field, and the block is skipped [S1, unofficial].
- **Full Clipped Field:** If a block is completely inside, the clipped field is "full". The implementation optimizes this by setting the field to `null` [S1, unofficial].

## Subtopics

### Voxel Body Carving with Sub-Voxel Clipping
This variant modifies the source grid in-place (or creates a modified copy) by removing the intersection of the volume with the bounding box. It uses `ClipBlock` with `keepInside: false` to get the remaining part of the block outside the box [S1, unofficial]. If the remainder is null or empty, the chunk block is set to Air. Otherwise, the chunk refinement is set to the remainder [S1, unofficial]. It also tracks touched chunks and their Y-range, marking the 3x3 neighborhood in XZ (dx, dz from -1 to 1) as dirty with the tracked Y-range to ensure face culling and ambient occlusion updates [S1, unofficial].

## Operating characteristics
**Time Complexity:** Linear with respect to the number of solid blocks in the source volume. Each block is visited once, and clipping is performed only on blocks intersecting the boundary.
**Memory Complexity:** Proportional to the number of solid blocks within the target bounding box. The output `VoxelBody` contains only the clipped blocks.
**Strengths:** Efficiently extracts sub-volumes without processing empty space. Preserves block-level structure, which is beneficial for chunk-based rendering and physics.
**Limitations:** Does not support arbitrary voxel-level precision; blocks are clipped at their boundaries. If the bounding box cuts through many blocks, the number of output voxels can be large.
**When to Use:** When extracting a specific region of a voxel world for editing, saving, or rendering.
**When Not to Use:** When the entire grid needs to be duplicated (use [Grid Clone](7232-grid-clone.md)) or when voxel-level precision is required (use [Bounding Box Clipping](4643-bounding-box-clipping.md)).

## Background
This method is part of the VoxelTerraformer engine's voxel manipulation toolkit. It is designed to handle the common task of extracting or modifying regions of a voxel world while maintaining the efficiency of block-based storage. The approach leverages the fact that voxel worlds are often composed of discrete solid blocks, allowing for coarser clipping operations than per-voxel methods.

## Sources
[S1] GhostlyActive/VoxelTerraformer, https://github.com/GhostlyActive/VoxelTerraformer, fetched 2026-10-27 (unofficial)
