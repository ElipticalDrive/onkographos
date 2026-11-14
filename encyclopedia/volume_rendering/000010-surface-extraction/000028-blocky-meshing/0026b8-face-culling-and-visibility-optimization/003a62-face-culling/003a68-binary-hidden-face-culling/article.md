# Binary Hidden-Face Culling

Also called: Bitmask Hidden-Face Culling, Visible-Face Culling

## Overview
Binary Hidden-Face Culling is a mesh generation optimization that reduces the number of faces in a voxel mesh by removing those completely hidden by adjacent voxels, using bitwise operations on a bitmask representation of voxels to determine visible faces in bulk [S2, unofficial]. It solves the performance bottleneck of naive neighbor-checking loops by replacing random memory access with parallel bitwise math [S2, unofficial]. Unlike [Face Culling](8735-face-culling.md), which typically iterates through neighbor coordinates, this method operates on linear bitmasks to identify exposed surfaces without explicit spatial lookups.

## How it works
**Inputs and Outputs:**
- Input: A linear bitmask $M$ representing a 1D slice or flattened array of voxels, where bit $i$ is 1 if voxel $i$ is solid and 0 if it is transparent/air.
- Output: Four bitmasks ($L, R, F, B$) indicating which voxels have visible Left, Right, Front, and Back faces, respectively. (Top/Bottom faces are handled similarly if the data is 3D, but the core logic applies to any axis).

**Data Structures:**
- **Bitmask ($M$):** A contiguous integer or array of integers. Each bit corresponds to a voxel position in a specific axis-aligned line.
  - Bit value `1`: Solid voxel.
  - Bit value `0`: Empty/Air voxel.

**Steps:**
1. **Initialize Mask:** Create bitmask $M$ where each bit represents the solid/empty state of a voxel along a chosen axis.
2. **Calculate Left-Face Visibility:**
   - Shift $M$ right by 1 position: $M_{right} = M \gg 1$. This aligns the neighbor to the left.
   - Invert the result: $\neg M_{right}$. This identifies positions where the left neighbor is air.
   - Bitwise AND with original mask: $L = M \ \& \ (\neg M_{right})$.
   - Result $L$ has bit $i$ set if voxel $i$ is solid AND voxel $i-1$ is air.
3. **Calculate Right-Face Visibility:**
   - Shift $M$ left by 1 position: $M_{left} = M \ll 1$. This aligns the neighbor to the right.
   - Invert the result: $\neg M_{left}$. This identifies positions where the right neighbor is air.
   - Bitwise AND with original mask: $R = M \ \& \ (\neg M_{left})$.
   - Result $R$ has bit $i$ set if voxel $i$ is solid AND voxel $i+1$ is air.
4. **Repeat for Other Axes:** Apply the same shift-and-invert logic for Front/Back and Top/Bottom axes if processing 3D data in a flattened structure or multiple 1D slices.
5. **Mesh Generation:** Use the resulting masks ($L, R, F, B, T, B$) to determine which faces to include in the final mesh geometry. If a bit is set in a mask, generate the corresponding face for that voxel.

**Formulas:**
For a given axis, let $M$ be the voxel state bitmask.
$$ L = M \ \& \ (\neg (M \gg 1)) $$
$$ R = M \ \& \ (\neg (M \ll 1)) $$
Where $\gg$ is logical right shift, $\ll$ is logical left shift, $\neg$ is bitwise NOT, and $\&$ is bitwise AND.

**Parameters:**
- Bit width: Depends on the chunk size or array length. For a 16x16x16 chunk, 4096 bits are required per axis.

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Shifts introduce zeros at the boundaries. For the left face, the leftmost voxel (index 0) has no left neighbor; the shift introduces a 0, so $\neg 0 = 1$, meaning the leftmost solid voxel will always have its left face visible, which is correct. Similarly for the rightmost voxel and right faces.
- **Precision:** Ensure the integer type used for the bitmask is large enough to hold the entire slice. For large chunks, use arrays of integers or arbitrary-precision integers.
- **Performance:** The original approach of looping through neighbors and performing random memory access leads to very poor performance [S2, unofficial]. This method avoids that by using bulk bitwise operations.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_visible_faces(bitmask: int) -> tuple:
    """
    Calculates visible left and right faces for a 1D slice of voxels.
    
    Args:
        bitmask: Integer where bit i is 1 if voxel i is solid, 0 otherwise.
        
    Returns:
        tuple: (left_visible_mask, right_visible_mask)
    """
    # Calculate left faces: solid voxel with air to the left
    # Shift right to align with left neighbor
    left_neighbor_mask = bitmask >> 1
    # Invert to find air positions
    air_left_mask = ~left_neighbor_mask
    # Solid voxels with air on the left are visible on the left face
    left_visible = bitmask & air_left_mask
    
    # Calculate right faces: solid voxel with air to the right
    # Shift left to align with right neighbor
    right_neighbor_mask = bitmask << 1
    # Invert to find air positions
    air_right_mask = ~right_neighbor_mask
    # Solid voxels with air on the right are visible on the right face
    right_visible = bitmask & air_right_mask
    
    return left_visible, right_visible
```

## Subtopics

### Chunk-Based Culling
Applying this method per chunk to reduce memory footprint and improve cache locality. Choose when memory bandwidth is a bottleneck. See [Chunk-Based Voxel Rendering](link).

### Greedy Meshing Integration
Combining binary culling with greedy meshing to further reduce vertex count by merging adjacent faces. Choose when rendering performance is critical. See [Greedy Meshing](link).

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, but with a very small constant factor due to bitwise parallelism. It is significantly faster than $O(N)$ neighbor-checking loops with poor cache behavior.
- **Memory Complexity:** $O(N)$ bits for the bitmask.
- **Measured Performance:** The original neighbor-checking approach suffered from poor performance due to random memory access [S2, unofficial]. Bitmask operations are cache-friendly and can be processed in parallel by CPU SIMD instructions.
- **Strengths:** Extremely fast for determining face visibility; avoids branching and random memory access.
- **Limitations:** Requires the voxel data to be accessible in a linear bitmask format. Does not handle occlusion from non-adjacent voxels (e.g., distant objects).
- **When to Use:** When generating voxel meshes on the CPU and performance is critical.
- **When Not to Use:** When occlusion culling from distant objects is required (use frustum or occlusion culling instead).

## Background
Binary Hidden-Face Culling was popularized in the context of voxel engine development, specifically in projects like "YNL - Vozel" [S2, unofficial]. It addresses the inefficiency of naive face culling by leveraging bitwise operations. The technique is also referenced in discussions about chunk-based voxel engines, such as "blockbound" [S1, unofficial].

## Sources
[S1] Phase 2: voxel engine com chunks, meshing e edição, https://github.com/EduardoLavall/blockbound/issues/4, fetched 2026-10-26 (unofficial)
[S2] Devlog #1.3: Hidden-face Culling - YNL, https://yunasawa.itch.io/ynl-vozel/devlog/1028290/devlog-13-hidden-face-culling, fetched 2026-10-27 (unofficial)
