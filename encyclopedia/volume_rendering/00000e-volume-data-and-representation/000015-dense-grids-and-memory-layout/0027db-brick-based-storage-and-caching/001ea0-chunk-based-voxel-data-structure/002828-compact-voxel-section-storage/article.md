# Compact Voxel Section Storage

Also called: Palette-based Voxel Storage, SectionStorage, Sparse Voxel Section Storage with Palette Indexing

## Overview
Compact Voxel Section Storage is a memory-optimized representation for a fixed-size 3D block of voxels (a "section") that dynamically selects between three storage modes—Empty, Uniform, and Palette—to minimize heap usage based on content entropy [S1, unofficial]. It sets itself apart from sibling methods like [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by maintaining a dense grid layout for locality while using palette indexing to compress mixed-material sections, and differs from [Chunk-based World Storage](2858-chunk-based-world-storage.md) by operating at the finer granularity of a 16³ section rather than a larger world chunk.

## How it works
**Inputs and Outputs:**
- Input: An array of 4096 `VoxelId` values (application-defined material identifiers, where 0 is air).
- Output: A `SectionStorage` structure that supports O(1) read/write access to any voxel in the 16×16×16 section.

**Data Structures:**
The core structure `SectionStorage` contains a single field `data` of type `Representation`, an enum with three variants:

1.  `Representation::Empty`: Holds no data. Used when all 4096 voxels are air (ID 0).
2.  `Representation::Uniform`: Holds a single `VoxelId`. Used when all 4096 voxels share the same non-zero ID.
3.  `Representation::Palette`: Holds a `Palette` struct:
    -   `entries`: A `Vec<PaletteEntry>`. Each `PaletteEntry` is a struct with two fields: `id` (4 bytes, `VoxelId`) and `count` (2 bytes, `u16`). Total size per entry is 6 bytes, packed sequentially.
    -   `words`: A `Vec<u64>` containing the bit-packed palette indices.
    -   `bits`: A `u8` indicating the number of bits used to store each index (1–12).

**Local Coordinate Mapping:**
Voxels are addressed by a linear index `i` from 0 to 4095. This maps to 3D coordinates $(x, y, z)$ where $x, y, z \in [0, 15]$ using:
$$ x = i \pmod{16} $$
$$ z = \lfloor i / 16 \rfloor \pmod{16} $$
$$ y = \lfloor i / 256 \rfloor $$
The linear index is calculated as:
$$ i = y \cdot 256 + z \cdot 16 + x $$

**Parameters:**
-   `SECTION_VOLUME`: Constant 4096.
-   `bits`: Calculated as $\lceil \log_2(\text{unique\_ids}) \rceil$. Minimum 1, maximum 12 (since $2^{12} = 4096$).

**Algorithm Steps:**

### 1. Initialization (`from_voxels`)
Constructs a `SectionStorage` from a raw array of 4096 `VoxelId`s.

1.  Validate input length is 4096.
2.  Check if all voxels are 0. If so, return `Representation::Empty`.
3.  Check if all voxels are identical. If so, return `Representation::Uniform` with that ID.
4.  Initialize an empty `entries` vector and a stack-allocated `indices` array of size 4096.
5.  Iterate `i` from 0 to 4095:
    a. Let `id = blocks[i]`.
    b. Search `entries` for an entry with `id`.
    c. If found, increment `entry.count`.
    d. If not found, append `PaletteEntry { id, count: 1 }` to `entries` and record its new index as `slot`.
    e. Store `slot` in `indices[i]`.
6.  Shrink `entries` to remove any capacity.
7.  Calculate `bits = index_bits(entries.len())`.
8.  Calculate `word_count = (4096 * bits + 63) / 64`.
9.  Allocate `words` as a vector of `u64` of size `word_count`, initialized to 0.
10. Iterate `i` from 0 to 4095:
    a. Call `write_index(words, bits, i, indices[i])`.
11. Return `SectionStorage { data: Representation::Palette { entries, words, bits } }`.

### 2. Read (`at_index`)
Returns the `VoxelId` at linear index `i`.

1.  Match on `data`:
    -   If `Empty`, return `VoxelId::AIR` (0).
    -   If `Uniform`, return the stored `id`.
    -   If `Palette`:
        a. Call `slot = read_index(palette.words, palette.bits, i)`.
        b. Return `palette.entries[slot].id`.

### 3. Bit-Packed Read (`read_index`)
Extracts a multi-bit value from the `words` array.

1.  Calculate `bit_pos = i * bits`.
2.  Calculate `word_idx = bit_pos / 64`.
3.  Calculate `offset = bit_pos % 64`.
4.  Extract `low = words[word_idx] >> offset`.
5.  If `offset + bits > 64`:
    a. Extract `high = words[word_idx + 1] << (64 - offset)`.
    b. `value = low | high`.
6.  Else:
    a. `value = low`.
7.  Mask `value` with `(1 << bits) - 1`.
8.  Return `value`.

### 4. Bit-Packed Write (`write_index`)
Sets a multi-bit value in the `words` array.

1.  Calculate `bit_pos = i * bits`.
2.  Calculate `word_idx = bit_pos / 64`.
3.  Calculate `offset = bit_pos % 64`.
4.  Calculate `mask = (1 << bits) - 1`.
5.  Clear bits in `words[word_idx]`:
    a. `words[word_idx] &= !(mask << offset)`.
    b. `words[word_idx] |= (value << offset) & (mask << offset)`.
6.  If `offset + bits > 64`:
    a. Calculate `high_bits = offset + bits - 64`.
    b. Calculate `high_mask = (1 << high_bits) - 1`.
    c. Clear high bits in `words[word_idx + 1]`:
        i. `words[word_idx + 1] &= !high_mask`.
        ii. `words[word_idx + 1] |= (value >> (64 - offset)) & high_mask`.

### 5. Bulk Replacement (`replace_from_voxels`)
Updates the storage with new voxel data, reusing buffers where possible.

1.  Validate input length is 4096.
2.  If all voxels are identical:
    a. If `data` is `Uniform`, update ID.
    b. If `data` is `Empty` or `Palette`, replace `data` with `Uniform(id)`.
    c. Return.
3.  If all voxels are 0:
    a. Replace `data` with `Empty`.
    b. Return.
4.  Ensure `data` is `Palette`. If not, allocate new `Palette`.
5.  Clear `entries`.
6.  Use stack-allocated `indices` array.
7.  Iterate `blocks`:
    a. Find or create `PaletteEntry` in `entries`.
    b. Increment count.
    c. Store slot in `indices`.
8.  Calculate new `bits`.
9.  Resize `words` to `word_count(new_bits)` and fill with 0.
10. Write indices to `words`.

### 6. Compaction (`compact`)
Reduces memory usage by removing unused palette slots and shrinking bit width.

1.  If `data` is not `Palette`, return false.
2.  Count `live` entries where `count != 0`.
3.  If `live == entries.len()` and `entries.capacity() == entries.len()`, return false (already compact).
4.  If `live < entries.len()`:
    a. Create `mapping` array of size `entries.len()`.
    b. Iterate `entries`: if `count != 0`, set `mapping[old_slot] = new_slot` and push entry to new list.
    c. Calculate `new_bits = index_bits(live)`.
    d. Allocate new `words`.
    e. Iterate `i` from 0 to 4095:
        i. `old_slot = read_index(old_words, old_bits, i)`.
        ii. `new_slot = mapping[old_slot]`.
        iii. `write_index(new_words, new_bits, i, new_slot)`.
    f. Replace `palette` with new entries, words, and bits.
5.  Else (only shrink capacity, no live count change):
    a. `entries.shrink_to_fit()`.
6.  Return true.

**Edge Cases and Pitfalls:**
-   **Cross-word boundary:** When `offset + bits > 64`, the read/write must combine bits from `words[word]` and `words[word+1]` to avoid corruption [S1, unofficial].
-   **Palette Exhaustion:** If a new ID is added to a full palette (4096 unique IDs, 12 bits), the code checks for unused slots (`count == 0`). If none exist, it triggers a repack to increase bit width, though with 12 bits, the palette is already at max capacity for 4096 slots [S1, unofficial].
-   **High Entropy:** If all 4096 voxels are unique, storage is ~38,912 bytes, which is larger than a dense `[u32; 4096]` array (16,384 bytes). The method does not automatically fall back to dense storage [S1, unofficial].

## Subtopics

### [Palette Compression with Reference Counting and Reuse](14211-palette-compression-with-reference-counting-and-reuse.md)
Palette Compression with Reference Counting and Reuse is a memory optimization technique for dense voxel grids that maps voxel types to a compact palette of entries, storing indices into the palette rather than full type values to save memory. It sets itself apart from sibling methods in the [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) tree by specifically handling mixed-material sections through dynamic palette management with reference counting, rather than relying on static empty or uniform modes.

### Voxel Color Indexing
Voxel Color Indexing is a technique where 3D textures store references to indexed colors held in a separate texture, saving storage by storing only a single color channel per voxel [S2, unofficial]. It reduces storage by ~75% per voxel [S2, unofficial]. The data structure consists of two textures: a 3D texture containing monochrome index values, and a separate 2D texture containing the palette of colors mapped to those indices [S2, unofficial]. A pitfall is that visualizing 3D textures in 2D is difficult [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Read and Write operations are O(1) time, involving bit shifts and masks on 64-bit integers [S1, unofficial].
-   **Memory Complexity:**
    -   Empty/Uniform: O(1) bytes (no heap allocation).
    -   Palette: O(P + 4096*B/8) bytes, where P is palette size (6 bytes per entry) and B is bits per index [S1, unofficial].
-   **Measured Performance:** On an AMD Ryzen 7 5800H, a section with 4,096 unique solid IDs uses 38,912 bytes of heap storage, compared to 0 bytes for Empty/Uniform and 1,048 bytes for synthetic terrain [S1, unofficial]. A dense `[u32; 4096]` array uses 16,384 bytes. High-entropy palettes can consume more memory than dense storage [S1, unofficial].
-   **Strengths:** Significant memory savings for low-entropy sections (Empty, Uniform, or few materials). No heap allocation for Empty/Uniform modes.
-   **Limitations:** High-entropy sections (many unique materials) use more memory than a dense array. No automatic fallback to dense storage.
-   **When to Use:** Ideal for voxel worlds with large areas of air, uniform blocks, or limited material variety per section.
-   **When Not to Use:** When sections are expected to have high entropy (many unique materials per section) and memory overhead of palette management is unacceptable.

## Background
Compact Voxel Section Storage was extracted and adapted from a local Rust/Vulkan voxel engine called `voxel-forge` [S1, unofficial]. It is designed to support greedy meshing and neighbor access for efficient terrain rendering. The method is part of a broader system that includes coordinate handling for infinite voxel worlds and meshing optimizations.

## Sources
[S1] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
[S2] Teardown Render Techniques - GitHub Pages, https://zacxalot.github.io/rendering/9-teardown/, fetched 2026-10-29 (unofficial)
