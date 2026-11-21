# Palette Compression with Reference Counting and Reuse

Also called: Palette Storage, Voxel Palette

## Overview
Palette Compression with Reference Counting and Reuse is a memory optimization technique for dense voxel grids that maps voxel types to a compact palette of entries, storing indices into the palette rather than full type values to save memory [S1]. It sets itself apart from sibling methods in the [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) tree by specifically handling mixed-material sections through dynamic palette management with reference counting, rather than relying on static empty or uniform modes [G].

## How it works
**Inputs and Outputs:**
- Input: A voxel position within a chunk, the new `voxel_type_id` to assign, and the current state of the palette and index arrays [S1].
- Output: Updated palette and index arrays reflecting the new voxel type assignment [S1].

**Data Structures:**
The storage consists of two parallel arrays [S1]:
1. `palette`: An array of `PaletteEntry` structs [S1].
    - `PaletteEntry` layout:
        - `voxel_type_id`: The identifier for the voxel type [S1].
        - `refcount`: An integer tracking the number of voxels currently referencing this palette entry [S1].
2. `indices`: An array mapping voxel positions to palette indices [S1].
    - Type: `byte[]` is recommended if the palette size is kept below 256 variants to save memory compared to `short[]` or `int[]` [S1].
    - Size: Equal to the volume size of the chunk [S1].

**Chunk Constants:**
- `EDGE_SIZE`: 32 [S1].
- `VOLUME_SIZE`: $32 \times 32 \times 32 = 32,768$ [S1].

**Parameters:**
- Palette size limit: Keeping the palette below 256 variants allows the use of plain bytes for indices, which is viable and saves memory compared to using a volume of shorts [S1].

**Numbered Steps:**
1. Calculate the linear index of the voxel position within the chunk [S1].
2. Retrieve the current palette index for this position: `old_palette_id = indices[index]` [S1].
3. Decrement the reference count of the old palette entry: `palette[old_palette_id].refcount -= 1` [S1].
4. Search for an existing entry in the palette matching the target `voxel_type_id` using `Array.FindIndex` (or equivalent linear search) [S1].
5. If an existing entry is found (`replace != -1`):
    - Update the voxel index to point to this existing entry: `indices[index] = replace` [S1].
    - Increment the reference count of the found entry: `palette[replace].refcount += 1` [S1].
    - Return [S1].
6. If no existing entry is found:
    - Check if the `old_palette_id` entry has a `refcount` of 0 [S1].
    - If `palette[old_palette_id].refcount == 0`:
        - Overwrite the old entry with the new `voxel_type_id` and set its `refcount` to 1 [S1].
        - Do not change the index in the volume array, as it already points to this slot [S1].
        - Return [S1].
    - If no unused slot is available:
        - Grow the palette array by doubling its size using `Array.Resize` with a minimum size of 4 elements, calculated as `Math.max(palette.Length, 2) * 2` [S1].
        - Add the new entry to the last position of the resized palette, set its `refcount` to 1 [S1].
        - Update the voxel index to point to this new last position [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class PaletteEntry:
    def __init__(self, voxel_type_id=None, refcount=0):
        self.voxel_type_id = voxel_type_id
        self.refcount = refcount

def set_voxel_type(palette, indices, index, voxel_type_id):
    """
    Sets the voxel type at a given linear index in a chunk.
    """
    # Step 2: Get old palette ID
    old_palette_id = indices[index]

    # Step 3: Decrement refcount of old entry
    palette[old_palette_id].refcount -= 1

    # Step 4: Search for existing entry
    replace = -1
    for i, entry in enumerate(palette):
        if entry.voxel_type_id == voxel_type_id:
            replace = i
            break

    # Step 5: If found, reuse
    if replace != -1:
        indices[index] = replace
        palette[replace].refcount += 1
        return

    # Step 6: If not found, try to reuse unused old slot or grow
    if palette[old_palette_id].refcount == 0:
        # Reuse the old slot
        palette[old_palette_id].voxel_type_id = voxel_type_id
        palette[old_palette_id].refcount = 1
        # indices[index] remains old_palette_id
        return

    # No unused slot, grow palette
    # Double size, minimum 4 elements via Math.max(palette.Length, 2) * 2
    new_size = max(len(palette), 2) * 2
    # Extend palette with zero-initialized entries
    palette.extend([PaletteEntry() for _ in range(new_size - len(palette))])

    last = len(palette) - 1
    palette[last].voxel_type_id = voxel_type_id
    palette[last].refcount = 1
    indices[index] = last
```

**Edge Cases and Pitfalls:**
- **Index Compression:** The initial implementation may forget to compress the indices [S1]. Compressing indices is optional, and using plain bytes is acceptable if the palette stays below 256 variants [S1].
- **Refcount Underflow:** Ensure that `refcount` is not decremented below zero [G]. This should not happen if every `set` operation correctly decrements the old entry's refcount before incrementing the new one [G].
- **Palette Growth:** When growing the palette, ensure that new entries are properly initialized with zero refcount to avoid stale data issues [S1].

## Subtopics

### Block Type Indexing with Registry
A memory optimization technique where block data is stored as compact type indices rather than full definitions to reduce RAM usage from ~256 bytes per block to 1 byte [S2, unofficial]. It uses a `BlockType` enum stored in a flat array inside the Chunk class, alongside a `BlockDef` struct registry for properties [S2, unofficial]. Using a hash map for block property lookup can become a major bottleneck during meshing, making a simple array preferable [S2, unofficial].

### Generic Packed Storage Interface
An abstraction (`IPackedStorage<T>`) that allows the palette storage to be reused for additional layers like simulated liquids, cellular-automata lighting, or per-voxel color tints [S1]. It is generic over equatable struct elements to overlay data atop base voxel volumes [S1].

## Operating characteristics
- **Time Complexity:** Lookup/Update is $O(P)$ where $P$ is the number of unique voxel types in the palette, due to the linear search (`Array.FindIndex`) [S1]. Palette growth is $O(P)$ for resizing and copying, but amortized $O(1)$ if growth is infrequent [G].
- **Memory Complexity:** The palette requires $O(P)$ entries, where each entry is the size of `voxel_type_id` plus an integer `refcount` [S1]. Indices require $O(V)$ bytes, where $V$ is the volume size (32,768 for a 32^3 chunk), assuming byte indices [S1].
- **Measured Performance:** Not given in the sources [S1].
- **Scaling:** Performance degrades linearly with the number of unique voxel types due to the linear search [S1]. Keeping the palette small (<256) is critical for memory efficiency [S1].
- **Strengths:** Simple implementation, effective memory savings for scenes with limited material variety, reusable via generic interfaces [S1].
- **Limitations:** Linear search time for palette lookups, potential memory overhead if palette grows large and indices must be widened [S1].
- **When to Use:** When voxel types are limited and memory is constrained, and the scene allows for dynamic palette management [G].
- **When Not to Use:** When the number of unique voxel types is very large, leading to palette bloat and index widening, or when lookup speed is critical and a hash map or direct indexing is preferred [G].

## Background
This method is described in a discussion on palette compression for voxel chunks [S1]. It builds on the concept of palettes in computing, where a small set of colors or types is mapped to indices [S1]. The original article was published on longor.net [S1].

## Sources
[S1] Palette Compression, https://voxel.wiki/wiki/palette-compression, fetched 2026-10-27
[S2] How I've optimized chunk generation in my Minecraft clone, https://dev.to/mainr/how-ive-optimized-chunk-generation-in-my-minecraft-clone-i78, fetched 2026-10-28 (unofficial)
