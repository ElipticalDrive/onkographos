# Compact Voxel Section Storage

Also called: Palette-based Voxel Storage, SectionStorage, Sparse Voxel Section Storage with Palette Indexing

## Overview
Compact Voxel Section Storage is a memory-optimized representation for a fixed-size $16 \times 16 \times 16$ voxel section (4096 voxels) that minimizes heap allocation by distinguishing between empty, uniform, and sparse material distributions [S1, unofficial]. Unlike dense arrays which always consume fixed memory, this method uses bit-packed palette indices for mixed materials, achieving $O(1)$ bytes for trivial sections and compact storage for low-entropy terrain, while supporting $O(1)$ random access [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input**: A sequence of 4096 `VoxelId` values (32-bit integers, where 0 represents air) [S1, unofficial].
*   **Output**: A `SectionStorage` object containing either no heap data (Empty/Uniform) or a palette of unique IDs and a bit-packed array of indices (Palette) [S1, unofficial].

### Data Structures
**`SectionStorage`**
Contains a single field `data` of type `Representation` [S1, unofficial].
*   `Representation::Empty`: No heap allocation [S1, unofficial].
*   `Representation::Uniform(VoxelId)`: Holds a single `VoxelId` [S1, unofficial].
*   `Representation::Palette(Palette)`: Holds a `Palette` struct [S1, unofficial].

**`Palette`**
*   `entries`: `Vec<PaletteEntry>`. A dynamic array of unique material IDs found in the section [S1, unofficial].
*   `words`: `Vec<u64>`. A bit-packed array storing the palette indices for each of the 4096 voxels [S1, unofficial].
*   `bits`: `u8`. The number of bits used to store each index in `words` [S1, unofficial].

**`PaletteEntry`**
*   `id`: `VoxelId` (4 bytes). The material identifier [S1, unofficial].
*   `count`: `u16` (2 bytes). The number of voxels in the section with this ID [S1, unofficial].
*   Total size: 6 bytes, packed sequentially [S1, unofficial].

**Constants**
*   `SECTION_VOLUME`: 4096 ($16^3$) [S1, unofficial].
*   `SECTION_EDGE`: 16 [S1, unofficial].

### Parameters
*   `bits`: The bit-width for palette indices, calculated based on the number of unique materials (`entries.len()`) [S1, unofficial]. Range is 1 to 12 bits [S1, unofficial]. If `entries.len()` exceeds $2^{12}$, the storage may become less efficient than dense storage, but no automatic fallback is implemented [S1, unofficial].

### Algorithm Steps

#### 1. Initialization (`from_voxels`)
Constructs a `SectionStorage` from a raw array of 4096 `VoxelId`s [S1, unofficial].
1.  Validate that the input length equals `SECTION_VOLUME` [S1, unofficial].
2.  Check if all blocks are identical; if so, return `Uniform` or `Empty` [S1, unofficial].
3.  Iterate blocks to build palette: find existing entry by ID or push new `PaletteEntry` with count 0 [S1, unofficial].
4.  Increment count for the found/created entry [S1, unofficial].
5.  Store the palette index (slot) in a stack-allocated `indices` array [S1, unofficial].
6.  Shrink palette entries vector [S1, unofficial].
7.  Calculate `bits` using `index_bits(entries.len())` [S1, unofficial].
8.  Allocate `words` vector of `u64` with size from `word_count(bits)` [S1, unofficial].
9.  Write indices into words using `write_index` [S1, unofficial].

#### 2. Bulk Replacement (`replace_from_voxels`)
Updates the storage with new voxel data, reusing buffers where possible [S1, unofficial].
1.  Validate `blocks.len() == SECTION_VOLUME` [S1, unofficial].
2.  If all blocks are identical, call `fill(first)` [S1, unofficial].
3.  Otherwise, ensure `data` is `Representation::Palette` [S1, unofficial].
4.  Clear `entries` [S1, unofficial].
5.  Use a stack-allocated `indices` array `[0_u16; SECTION_VOLUME]` [S1, unofficial].
6.  Iterate `blocks`: find slot in `entries` or push new `PaletteEntry { id, count: 0 }` [S1, unofficial].
7.  Increment count [S1, unofficial].
8.  Store slot in `indices` [S1, unofficial].
9.  Calculate new `bits` [S1, unofficial].
10. Resize `words` to `word_count(bits)` and fill with 0 [S1, unofficial].
11. Write indices to `words` [S1, unofficial].

#### 3. Read (`at_index`)
Returns the `VoxelId` at a local linear index $i$ ($0 \le i < 4096$) [S1, unofficial].
1.  Match on `data` [S1, unofficial].
2.  If `Empty`, return `VoxelId::AIR` [S1, unofficial].
3.  If `Uniform`, return `*id` [S1, unofficial].
4.  If `Palette`, read index from `words` using `read_index(palette.words, palette.bits, index)`, then return `palette.entries[slot].id` [S1, unofficial].

#### 4. Bit-Packed Read (`read_index`)
Extracts a multi-bit value from the `words` array [S1, unofficial].
1.  Calculate `bit = index * bits` [S1, unofficial].
2.  Calculate `word = bit / 64` [S1, unofficial].
3.  Calculate `offset = bit % 64` [S1, unofficial].
4.  Extract `value = words[word] >> offset` [S1, unofficial].
5.  If `offset + bits > 64`, OR with `words[word + 1] << (64 - offset)` [S1, unofficial].
6.  Mask result with `(1 << bits) - 1` [S1, unofficial].

#### 5. Bit-Packed Write (`write_index`)
Writes a multi-bit value into the `words` array [S1, unofficial].
1.  Calculate `bit`, `word`, `offset` [S1, unofficial].
2.  Calculate `mask = (1 << bits) - 1` [S1, unofficial].
3.  Clear bits in `words[word]` using `!(mask << offset)` [S1, unofficial].
4.  OR with `value << offset` [S1, unofficial].
5.  If `offset + bits > 64`, calculate `high_bits = offset + bits - 64` [S1, unofficial].
6.  Calculate `high_mask = (1 << high_bits) - 1` [S1, unofficial].
7.  Clear high bits in `words[word + 1]` using `!high_mask` [S1, unofficial].
8.  OR with `value >> (64 - offset)` [S1, unofficial].

#### 6. Repacking (`repack`)
Increases the bit-width when the palette grows [S1, unofficial].
1.  Allocate `words` of size `word_count(new_bits)` [S1, unofficial].
2.  Iterate `index` from 0 to `SECTION_VOLUME` [S1, unofficial].
3.  Read old value using `read_index(old, old_bits, index)` [S1, unofficial].
4.  Write new value to new words using `write_index` with `new_bits` [S1, unofficial].

#### 7. Compaction (`compact`)
Reduces memory usage by removing unused palette slots and shrinking the word array [S1, unofficial].
1.  If not `Palette`, return false [S1, unofficial].
2.  Count `live` entries where `count != 0` [S1, unofficial].
3.  If `live == entries.len()` and `capacity == len()`, return false [S1, unofficial].
4.  If `has_unused` (live < len): Create `mapping` array [S1, unofficial].
5.  Iterate `entries`: if `count != 0`, set `mapping[old_slot] = entries.len()` and push entry [S1, unofficial].
6.  Calculate `bits = index_bits(live)` [S1, unofficial].
7.  Allocate new `words` [S1, unofficial].
8.  Iterate `index` 0 to `SECTION_VOLUME`: read old slot, write `mapping[slot]` to new words [S1, unofficial].
9.  Replace `palette` [S1, unofficial].
10. Else (only shrink capacity): `shrink_to_fit` entries [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

SECTION_VOLUME = 4096

def index_bits(num_entries: int) -> int:
    if num_entries <= 1:
        return 1
    return math.ceil(math.log2(num_entries))

def word_count(bits: int) -> int:
    return (SECTION_VOLUME * bits + 63) // 64

def read_index(words: list, bits: int, index: int) -> int:
    bit = index * bits
    word = bit // 64
    offset = bit % 64
    value = words[word] >> offset
    if offset + bits > 64:
        value |= words[word + 1] << (64 - offset)
    return value & ((1 << bits) - 1)

def write_index(words: list, bits: int, index: int, value: int):
    bit = index * bits
    word = bit // 64
    offset = bit % 64
    mask = (1 << bits) - 1
    words[word] &= ~(mask << offset)
    words[word] |= (value & mask) << offset
    if offset + bits > 64:
        high_bits = offset + bits - 64
        high_mask = (1 << high_bits) - 1
        words[word + 1] &= ~high_mask
        words[word + 1] |= (value >> (64 - offset)) & high_mask

class PaletteEntry:
    __slots__ = ('id', 'count')
    def __init__(self, id: int, count: int):
        self.id = id
        self.count = count

class Palette:
    def __init__(self, entries: list, words: list, bits: int):
        self.entries = entries
        self.words = words
        self.bits = bits

class SectionStorage:
    def __init__(self, data):
        self.data = data

    @staticmethod
    def from_voxels(voxels: list) -> 'SectionStorage':
        if len(voxels) != SECTION_VOLUME:
            raise ValueError("Invalid block count")
        if all(v == 0 for v in voxels):
            return SectionStorage("Empty")
        first = voxels[0]
        if all(v == first for v in voxels):
            return SectionStorage(("Uniform", first))
        
        entries = []
        indices = [0] * SECTION_VOLUME
        for i, v in enumerate(voxels):
            slot = None
            for s, e in enumerate(entries):
                if e.id == v:
                    slot = s
                    break
            if slot is None:
                entries.append(PaletteEntry(v, 0))
                slot = len(entries) - 1
            entries[slot].count += 1
            indices[i] = slot
            
        entries = entries[:len(entries)]
        bits = index_bits(len(entries))
        wc = word_count(bits)
        words = [0] * wc
        for i in range(SECTION_VOLUME):
            write_index(words, bits, i, indices[i])
        return SectionStorage(("Palette", Palette(entries, words, bits)))

    def at_index(self, index: int) -> int:
        if self.data == "Empty":
            return 0
        if self.data[0] == "Uniform":
            return self.data[1]
        palette = self.data[1]
        slot = read_index(palette.words, palette.bits, index)
        return palette.entries[slot].id
```

### Edge Cases and Pitfalls
*   **Cross-word bit boundary**: When `offset + bits > 64`, the index spans two `u64` words. The implementation must combine bits from `words[word]` and `words[word + 1]` to avoid data corruption [S1, unofficial].
*   **Palette slot exhaustion**: When adding a new ID to a full palette, the code checks if a slot with `count == 0` exists before allocating a new slot. If no free slot exists and the palette is full, it triggers a repack to increase bit width [S1, unofficial].
*   **High-entropy palettes**: High-entropy palettes (many unique materials) can consume more memory than a dense `[u32; 4096]` array (16,384 bytes). The library does not automatically fall back to dense storage [S1, unofficial].
*   **Coordinate handling**: World coordinates are split into section and local parts. Negative coordinates use floor division (Euclidean division) to correctly map to section indices. Overflow checks are required when computing neighbor offsets [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Read/Write**: $O(1)$ time complexity for accessing a single voxel, involving bit shifts and masks on 64-bit integers [S1, unofficial].
*   **Memory (Empty/Uniform)**: $O(1)$ bytes (no heap allocation) [S1, unofficial].
*   **Memory (Palette)**: $O(P + \frac{4096 \cdot B}{8})$ bytes, where $P$ is palette size and $B$ is bits per index [S1, unofficial].

### Measured Performance
On an AMD Ryzen 7 5800H (Windows 11, Rust 1.99):
*   **Empty Section**: 0 B heap storage [S1, unofficial].
*   **Uniform Section**: 0 B heap storage [S1, unofficial].
*   **Synthetic Terrain**: 1,048 B heap storage [S1, unofficial].
*   **Air/Solid Checkerboard**: 528 B heap storage [S1, unofficial].
*   **4,096 Unique Solid IDs**: 38,912 B heap storage (significantly larger than dense 16,384 B due to palette overhead) [S1, unofficial].

### Strengths and Limitations
*   **Strengths**: Extremely low memory usage for typical voxel worlds (mostly empty or uniform). Fast $O(1)$ access. Compatible with greedy meshing [S1, unofficial].
*   **Limitations**: High-entropy sections are memory-inefficient. No automatic fallback to dense storage for high entropy. Compaction requires iterating all voxels and repacking bits, which is costly if done frequently [S1, unofficial].

### When to Use
*   Use for voxel worlds with significant empty space or large uniform regions (e.g., terrain, caves) [S1, unofficial].
*   Do not use for highly detailed, high-entropy scenes (e.g., complex organic structures with many materials) where dense storage might be more compact [S1, unofficial].

## Background
Compact Voxel Section Storage is part of the `voxel-forge` library, an unofficial Rust/Vulkan voxel engine project [S1, unofficial]. The storage and coordinate code were extracted and adapted from this local engine [S1, unofficial]. The implementation focuses on minimizing heap allocations and supporting efficient meshing for real-time rendering [S1, unofficial].

## Sources

[S1] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
