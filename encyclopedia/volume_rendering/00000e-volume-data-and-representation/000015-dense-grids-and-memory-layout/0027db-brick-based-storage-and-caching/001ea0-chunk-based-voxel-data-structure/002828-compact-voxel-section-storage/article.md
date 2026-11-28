# Compact Voxel Section Storage

Also called: Palette-based Voxel Storage, SectionStorage, Sparse Voxel Section Storage with Palette Indexing

## Overview
Compact Voxel Section Storage is a memory-optimized representation for a fixed-size 3D block of voxels (a "section") that dynamically selects between three storage modes—Empty, Uniform, and Palette—to minimize heap usage based on content entropy [S1, unofficial]. It sets itself apart from sibling methods like [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by maintaining a dense grid layout for locality while using palette indexing to compress mixed-material sections, and differs from [Chunk-based World Storage](2858-chunk-based-world-storage.md) by operating at the finer granularity of a 16³ section rather than a larger world chunk [G].

## How it works
**Inputs and Outputs:**
- Input: An array of 4096 `VoxelId` values (application-defined material identifiers, where 0 is air) [S1, unofficial].
- Output: A `SectionStorage` structure that supports O(1) read/write access to any voxel in the 16×16×16 section [S1, unofficial].

**Data Structures:**
The core structure `SectionStorage` contains a single field `data` of type `Representation`, an enum with three variants:
1. `Representation::Empty` holds no data and is used when all 4096 voxels are air (ID 0) [S1, unofficial].
2. `Representation::Uniform` holds a single `VoxelId` and is used when all 4096 voxels share the same non-zero ID [S1, unofficial].
3. `Representation::Palette` holds a `Palette` struct containing `entries` (a `Vec<PaletteEntry>` where each entry has `id` at 4 bytes and `count` at 2 bytes, totaling 6 bytes packed sequentially), `words` (a `Vec<u64>` of bit-packed indices), and `bits` (a `u8` for the bit width per index, ranging from 1 to 12) [S1, unofficial].

**Local Coordinate Mapping:**
Voxels are addressed by a linear index `i` from 0 to 4095 [S1, unofficial]. This maps to 3D coordinates $(x, y, z)$ where $x, y, z \in [0, 15]$ using:
$$ x = i \pmod{16} $$
$$ z = \lfloor i / 16 \rfloor \pmod{16} $$
$$ y = \lfloor i / 256 \rfloor $$
The linear index is calculated as:
$$ i = y \cdot 256 + z \cdot 16 + x $$

**Parameters:**
- `SECTION_VOLUME`: Constant integer of 4096 [S1, unofficial].
- `bits`: Calculated as $\lceil \log_2(\text{unique\_ids}) \rceil$ with a range of 1 to 12 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
SECTION_VOLUME = 4096

class PaletteEntry:
    id: int  # 4 bytes
    count: int  # 2 bytes

class Palette:
    entries: list[PaletteEntry]
    words: list[int]  # u64
    bits: int  # u8

class SectionStorage:
    data: tuple  # ("Empty",) | ("Uniform", VoxelId) | ("Palette", Palette)

def index_bits(n: int) -> int:
    return max(1, (n - 1).bit_length())

def read_index(words: list[int], bits: int, index: int) -> int:
    bit_pos = index * bits
    word_idx = bit_pos // 64
    offset = bit_pos % 64
    value = words[word_idx] >> offset
    if offset + bits > 64:
        value |= words[word_idx + 1] << (64 - offset)
    return value & ((1 << bits) - 1)

def write_index(words: list[int], bits: int, index: int, value: int) -> None:
    bit_pos = index * bits
    word_idx = bit_pos // 64
    offset = bit_pos % 64
    mask = (1 << bits) - 1
    words[word_idx] &= ~(mask << offset)
    words[word_idx] |= value << offset
    if offset + bits > 64:
        high_bits = offset + bits - 64
        high_mask = (1 << high_bits) - 1
        words[word_idx + 1] &= ~high_mask
        words[word_idx + 1] |= (value >> (64 - offset)) & high_mask

def from_voxels(blocks: list[int]) -> SectionStorage:
    assert len(blocks) == SECTION_VOLUME
    if all(b == 0 for b in blocks):
        return SectionStorage(data=("Empty",))
    if all(b == blocks[0] for b in blocks):
        return SectionStorage(data=("Uniform", blocks[0]))
    entries = []
    indices = [0] * SECTION_VOLUME
    for i, id in enumerate(blocks):
        slot = None
        for j, entry in enumerate(entries):
            if entry.id == id:
                slot = j
                break
        if slot is None:
            entries.append(PaletteEntry(id=id, count=0))
            slot = len(entries) - 1
        entries[slot].count += 1
        indices[i] = slot
    entries = entries[:len(entries)]
    bits = index_bits(len(entries))
    word_count = (SECTION_VOLUME * bits + 63) // 64
    words = [0] * word_count
    for i in range(SECTION_VOLUME):
        write_index(words, bits, i, indices[i])
    return SectionStorage(data=("Palette", Palette(entries=entries, words=words, bits=bits)))

def at_index(storage: SectionStorage, index: int) -> int:
    data = storage.data
    if data[0] == "Empty":
        return 0
    if data[0] == "Uniform":
        return data[1]
    palette = data[1]
    slot = read_index(palette.words, palette.bits, index)
    return palette.entries[slot].id

def replace_from_voxels(storage: SectionStorage, blocks: list[int]) -> None:
    assert len(blocks) == SECTION_VOLUME
    if all(b == blocks[0] for b in blocks):
        storage.data = ("Uniform", blocks[0])
        return
    if all(b == 0 for b in blocks):
        storage.data = ("Empty",)
        return
    if storage.data[0] != "Palette":
        storage.data = ("Palette", Palette(entries=[], words=[], bits=0))
    palette = storage.data[1]
    palette.entries = []
    indices = [0] * SECTION_VOLUME
    for i, id in enumerate(blocks):
        slot = None
        for j, entry in enumerate(palette.entries):
            if entry.id == id:
                slot = j
                break
        if slot is None:
            palette.entries.append(PaletteEntry(id=id, count=0))
            slot = len(palette.entries) - 1
        palette.entries[slot].count += 1
        indices[i] = slot
    bits = index_bits(len(palette.entries))
    word_count = (SECTION_VOLUME * bits + 63) // 64
    palette.words = [0] * word_count
    for i in range(SECTION_VOLUME):
        write_index(palette.words, bits, i, indices[i])

def compact(storage: SectionStorage) -> bool:
    if storage.data[0] != "Palette":
        return False
    palette = storage.data[1]
    live = sum(1 for e in palette.entries if e.count != 0)
    if live == len(palette.entries):
        return False
    if live < len(palette.entries):
        mapping = [0] * len(palette.entries)
        new_entries = []
        for old_slot, entry in enumerate(palette.entries):
            if entry.count != 0:
                mapping[old_slot] = len(new_entries)
                new_entries.append(entry)
        bits = index_bits(len(new_entries))
        word_count = (SECTION_VOLUME * bits + 63) // 64
        new_words = [0] * word_count
        for i in range(SECTION_VOLUME):
            old_slot = read_index(palette.words, palette.bits, i)
            write_index(new_words, bits, i, mapping[old_slot])
        storage.data = ("Palette", Palette(entries=new_entries, words=new_words, bits=bits))
    else:
        palette.entries = palette.entries[:len(palette.entries)]
    return True
```

**Edge Cases and Pitfalls:**
- **Cross-word boundary:** When `offset + bits > 64`, the read/write must combine bits from `words[word]` and `words[word+1]` to avoid data corruption [S1, unofficial].
- **Palette slot exhaustion:** When adding a new ID to a full palette, the code checks for unused slots (`count == 0`) before allocating a new slot, and triggers a repack to increase bit width if none exist [S1, unofficial].
- **High entropy:** If all 4096 voxels are unique, storage consumes ~38,912 bytes, which exceeds a dense `[u32; 4096]` array (16,384 bytes), and the method does not automatically fall back to dense storage [S1, unofficial].

## Subtopics

### [Palette Compression with Reference Counting and Reuse](14211-palette-compression-with-reference-counting-and-reuse.md)
Palette Compression with Reference Counting and Reuse is a memory optimization technique for dense voxel grids that maps voxel types to a compact palette of entries, storing indices into the palette rather than full type values to save memory. It sets itself apart from sibling methods in the [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) tree by specifically handling mixed-material sections through dynamic palette management with reference counting, rather than relying on static empty or uniform modes [G].

### [Row-Packed Voxel Storage](20028-row-packed-voxel-storage.md)
Row-Packed Voxel Storage is a memory layout for voxel chunks that reduces memory overhead while maintaining constant-time row access by storing rows as either allocated (with an offset to uncompressed cell data) or unallocated (represented by a single material byte) [G].

### Voxel Color Indexing
Voxel Color Indexing is a technique where 3D textures store references to indexed colors held in a separate texture, saving storage by storing only a single color channel per voxel [S2, unofficial]. It reduces storage by ~75% per voxel [S2, unofficial]. The data structure consists of two textures: a 3D texture containing monochrome index values, and a separate 2D texture containing the palette of colors mapped to those indices [S2, unofficial]. A pitfall is that visualizing 3D textures in 2D is difficult [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Read and Write operations are O(1) time, involving bit shifts and masks on 64-bit integers [S1, unofficial].
- **Memory Complexity:** Empty and Uniform modes use O(1) bytes with no heap allocation [S1, unofficial]. Palette mode uses O(P + 4096*B/8) bytes, where P is palette size (6 bytes per entry) and B is bits per index [S1, unofficial].
- **Measured Performance:** On an AMD Ryzen 7 5800H, a section with 4,096 unique solid IDs uses 38,912 bytes of heap storage, compared to 0 bytes for Empty/Uniform and 1,048 bytes for synthetic terrain [S1, unofficial]. A dense `[u32; 4096]` array uses 16,384 bytes [S1, unofficial]. High-entropy palettes can consume more memory than dense storage [S1, unofficial].
- **Strengths:** Significant memory savings for low-entropy sections (Empty, Uniform, or few materials) [S1, unofficial].
- **Limitations:** High-entropy sections (many unique materials) use more memory than a dense array, and there is no automatic fallback to dense storage [S1, unofficial].
- **When to Use:** Ideal for voxel worlds with large areas of air, uniform blocks, or limited material variety per section [G].
- **When Not to Use:** When sections are expected to have high entropy (many unique materials per section) and memory overhead of palette management is unacceptable [G].

## Background
Compact Voxel Section Storage was extracted and adapted from a local Rust/Vulkan voxel engine called `voxel-forge` [S1, unofficial]. It is designed to support greedy meshing and neighbor access for efficient terrain rendering [S1, unofficial]. The method is part of a broader system that includes coordinate handling for infinite voxel worlds and meshing optimizations [S1, unofficial].

## Sources
[S1] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
[S2] Teardown Render Techniques - GitHub Pages, https://zacxalot.github.io/rendering/9-teardown/, fetched 2026-10-29 (unofficial)
