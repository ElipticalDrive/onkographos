# Voxel Interchange Format Serialization

Also called: voxel interchange format, voxel serialization

## Overview
Serializes a 3D voxel region into a flat 1D array accompanied by metadata for transport or storage [S1, unofficial]. It differs from its sibling [Binvox File Parsing](2862-binvox-file-parsing.md) by using a generic JSON-like structure with explicit dimensions and position rather than a binary header, and from [Hierarchical Sparse Voxel Format (glTF-based)](13799-hierarchical-sparse-voxel-format-gltf-based.md) by storing data in a flat 1D array rather than a hierarchical structure of volumes and primitives [S1, unofficial].

## How it works
**Inputs:** A voxel grid access function `getVoxelDataFor(x, y, z)` returning voxel data, and region parameters: `length`, `width`, `depth` (integers), and `start` (array `[startX, startY, startZ]`).
**Outputs:** A JSON object containing a packed 1D array of voxels, dimensions, and position.

**Data Structures:**
- `voxels`: An `Int8Array` (or equivalent 1D array) of size `length * width * depth`.
- `dimensions`: Array `[length, width, height]`.
- `position`: Array `[startX, startY, startZ]`.

**Steps:**
1. Initialize `length`, `width`, `depth`, and `start` coordinates.
2. Create an `Int8Array` of size `length * width * depth`.
3. Initialize `idx = 0`.
4. Loop `z` from `start[2]` to `depth`.
5. Loop `y` from `start[1]` to `height`.
6. Loop `x` from `start[0]` to `length`.
7. Inside the innermost loop, call `getVoxelDataFor(x+start[0], y+start[1], z+start[2])` and assign to `voxels[idx]`.
8. Increment `idx`.
9. Return the structure: `{ "voxels": [packed 1D array of voxels], "dimensions": [length, width, height], "position": [startX, startY, startZ] }`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def serialize_voxel_region(getVoxelDataFor, length, width, depth, start):
    # start is [startX, startY, startZ]
    # Note: The source snippet uses 'height' and 'depth' in loop bounds 
    # without defining them relative to 'length' and 'width'. 
    # We assume height=width and depth=depth for consistency with dimensions output.
    height = width
    # depth is already defined
    
    num_voxels = length * width * depth
    voxels = [0] * num_voxels
    idx = 0
    
    for z in range(start[2], depth):
        for y in range(start[1], height):
            for x in range(start[0], length):
                # The source snippet calls getVoxelDataFor with absolute coordinates
                # but adds start offsets again. This is likely a bug in the source
                # or implies getVoxelDataFor expects local coordinates.
                # We follow the source literally:
                data = getVoxelDataFor(x + start[0], y + start[1], z + start[2])
                voxels[idx] = data
                idx += 1
                
    return {
        "voxels": voxels,
        "dimensions": [length, width, height],
        "position": [start[0], start[1], start[2]]
    }
```

**Edge Cases and Pitfalls:**
- The provided loop bounds use `depth` and `height` variables that are not defined in the snippet, and `z`/`y`/`x` loop bounds mix absolute world coordinates with relative dimensions, which may cause out-of-bounds access or incorrect indexing if not adapted to local chunk coordinates [S1, unofficial].
- The coordinate calculation `x+start[0]` inside the loop where `x` starts at `start[0]` results in double-offsetting if `getVoxelDataFor` expects absolute coordinates. If `getVoxelDataFor` expects local coordinates, the `+start` should be removed.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the region.
- **Memory Complexity:** $O(N)$ for the 1D array.
- **Strengths:** Simple structure, easy to parse in most languages.
- **Limitations:** Flat array does not handle sparsity efficiently; potential for indexing errors due to ambiguous loop bounds in the reference implementation.

## Background
Originated in the Voxel.js ecosystem, specifically the `voxel-engine-stackgl` repository [S1, unofficial]. It is used for serializing voxel chunks for network transport or storage in voxel-based games.

## Sources
[S1] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
