# Compact Voxel Section Storage

Also called: Palette-based Voxel Storage, SectionStorage, Sparse Voxel Section Storage with Palette Indexing

## Overview
Compact Voxel Section Storage is a memory-optimized representation for a fixed-size $16 \times 16 \times 16$ voxel section (4096 voxels) that minimizes heap allocation by distinguishing between empty, uniform, and sparse material distributions [S1, unofficial]. Unlike dense arrays which always consume fixed memory, or complex tree structures, this method uses bit-packed palette indices for mixed materials, achieving $O(1)$ bytes for trivial sections and compact storage for low-entropy terrain, while supporting $O(1)$ random access [S1, unofficial]. It serves as the core data container within the Chunk-based Voxel Data Structure, providing the localized memory layout that enables efficient greedy meshing and neighbor access [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input**: A sequence of 4096 `VoxelId` values (32-bit integers, where 0 represents air).
*   **Output**: A `SectionStorage` object containing either no heap data (Empty/Uniform) or a palette of unique IDs and a bit-packed array of indices (Palette).

### Data Structures

**`SectionStorage`**
Contains a single field `data` of type `Representation`:
*   `Representation::Empty`: No heap allocation.
*   `Representation::Uniform(VoxelId)`: Holds a single `VoxelId`. No heap allocation for the ID itself (inline).
*   `Representation::Palette(Palette)`: Holds a `Palette` struct.

**`Palette`**
*   `entries`: `Vec<PaletteEntry>`. A dynamic array of unique material IDs found in the section.
*   `words`: `Vec<u64>`. A bit-packed array storing the palette indices for each of the 4096 voxels.
*   `bits`: `u8`. The number of bits used to store each index in `words`.

**`PaletteEntry`**
*   `id`: `VoxelId` (4 bytes). The material identifier.
*   `count`: `u16` (2 bytes). The number of voxels in the section with this ID.
*   Total size: 6 bytes, packed sequentially.

**Constants**
*   `SECTION_VOLUME`: 4096 ($16^3$).
*   `SECTION_EDGE`: 16.

### Parameters
*   `bits`: The bit-width for palette indices. Calculated as $\lceil \log_2(\text{entries.len()} + 1) \rceil$. The sources indicate a range of 1 to 12 bits [S1, unofficial]. If `entries.len()` exceeds $2^{12}$, the storage may become less efficient than dense storage, but no automatic fallback is implemented [S1, unofficial].

### Algorithm Steps

#### 1. Initialization (`from_voxels`)
Constructs a `SectionStorage` from a raw array of 4096 `VoxelId`s.

1.  Validate that the input length equals `SECTION_VOLUME`.
2.  Check if all voxels are identical:
    *   If all are 0 (air), return `Representation::Empty`.
    *   If all are the same non-zero ID, return `Representation::Uniform(id)`.
3.  Initialize an empty `entries` vector and a stack-allocated `indices` array of size 4096.
4.  Iterate through the input voxels:
    *   Search `entries` for an existing `PaletteEntry` with matching `id`.
    *   If found, increment its `count`.
    *   If not found, append a new `PaletteEntry { id, count: 1 }` to `entries`.
    *   Store the index (slot) of the found/created entry in `indices[i]`.
5.  Shrink `entries` to fit (remove any trailing capacity).
6.  Calculate `bits = index_bits(entries.len())`.
7.  Calculate `word_count = word_count(bits)`.
8.  Allocate `words` as a `Vec<u64>` of size `word_count`.
9.  Iterate $i$ from 0 to 4095:
    *   Call `write_index(words, bits, i, indices[i])`.
10. Return `Representation::Palette(Palette { entries, words, bits })`.

#### 2. Bulk Replacement (`replace_from_voxels`)
Updates the storage with new voxel data, reusing buffers where possible.

1.  Validate input length equals `SECTION_VOLUME`.
2.  Check if all voxels are identical:
    *   If all are 0, set `data = Representation::Empty`.
    *   If all are same `id`, set `data = Representation::Uniform(id)`.
    *   Otherwise, ensure `data` is `Palette`. If it is `Empty` or `Uniform`, allocate a new `Palette` structure.
3.  Clear `entries` (or reuse if capacity allows, but logic implies rebuilding for correctness in bulk replace).
4.  Use a stack-allocated `indices` array of size 4096.
5.  Iterate through input voxels:
    *   Find or create slot in `entries` (same logic as Initialization).
    *   Store slot index in `indices`.
6.  Calculate new `bits`.
7.  Resize `words` to `word_count(new_bits)` and fill with 0.
8.  Write indices to `words` using `write_index`.

#### 3. Read (`at_index`)
Returns the `VoxelId` at a local linear index $i$ ($0 \le i < 4096$).

1.  Match on `data`:
    *   If `Empty`, return `VoxelId::AIR` (0).
    *   If `Uniform`, return the stored `id`.
    *   If `Palette`:
        *   Call `slot = read_index(words, bits, i)`.
        *   Return `entries[slot].id`.

#### 4. Bit-Packed Read (`read_index`)
Extracts a multi-bit value from the `words` array.

1.  Calculate `bit = i * bits`.
2.  Calculate `word = bit / 64`.
3.  Calculate `offset = bit % 64`.
4.  Extract `low = words[word] >> offset`.
5.  If `offset + bits > 64`:
    *   Extract `high = words[word + 1] << (64 - offset)`.
    *   `value = low | high`.
6.  Else:
    *   `value = low`.
7.  Mask `value` with `(1 << bits) - 1`.
8.  Return `value`.

#### 5. Bit-Packed Write (`write_index`)
Writes a multi-bit value into the `words` array.

1.  Calculate `bit = i * bits`.
2.  Calculate `word = bit / 64`.
3.  Calculate `offset = bit % 64`.
4.  Calculate `mask = (1 << bits) - 1`.
5.  Clear bits in `words[word]`:
    *   `words[word] &= !(mask << offset)`.
6.  OR with new value:
    *   `words[word] |= (value & mask) << offset`.
7.  If `offset + bits > 64`:
    *   Calculate `high_bits = offset + bits - 64`.
    *   Calculate `high_mask = (1 << high_bits) - 1`.
    *   Clear high bits in `words[word + 1]`:
        *   `words[word + 1] &= !high_mask`.
    *   OR with high part of value:
        *   `words[word + 1] |= (value >> (64 - offset)) & high_mask`.

#### 6. Repacking
Increases the bit-width when the palette grows.

1.  Calculate `new_bits` based on new `entries.len()`.
2.  Allocate `new_words` of size `word_count(new_bits)`.
3.  Iterate $i$ from 0 to 4095:
    *   `old_val = read_index(old_words, old_bits, i)`.
    *   `write_index(new_words, new_bits, i, old_val)`.
4.  Replace `words` with `new_words` and update `bits`.

#### 7. Compaction (`compact`)
Reduces memory usage by removing unused palette slots and shrinking the word array.

1.  If `data` is not `Palette`, return `false`.
2.  Count `live` entries where `count != 0`.
3.  If `live == entries.len()` and `entries.capacity() == entries.len()`, return `false` (already compact).
4.  If `live < entries.len()` (holes exist):
    *   Create `mapping` array of size `entries.len()`.
    *   Iterate `entries`: if `count != 0`, set `mapping[old_slot] = new_slot` and push entry to new list.
    *   Calculate `new_bits = index_bits(live)`.
    *   Allocate `new_words` of size `word_count(new_bits)`.
    *   Iterate $i$ from 0 to 4095:
        *   `old_slot = read_index(words, bits, i)`.
        *   `new_slot = mapping[old_slot]`.
        *   `write_index(new_words, new_bits, i, new_slot)`.
    *   Replace `entries` and `words` with new structures.
5.  Else (only shrink capacity, no holes):
    *   `entries.shrink_to_fit()`.
6.  Return `true`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

SECTION_VOLUME = 4096
SECTION_EDGE = 16

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
    
    low = words[word] >> offset
    
    if offset + bits > 64:
        high = words[word + 1] << (64 - offset)
        value = low | high
    else:
        value = low
        
    mask = (1 << bits) - 1
    return value & mask

def write_index(words: list, bits: int, index: int, value: int):
    bit = index * bits
    word = bit // 64
    offset = bit % 64
    
    mask = (1 << bits) - 1
    
    # Clear bits in current word
    words[word] &= ~(mask << offset)
    # Set bits in current word
    words[word] |= (value & mask) << offset
    
    if offset + bits > 64:
        high_bits = offset + bits - 64
        high_mask = (1 << high_bits) - 1
        # Clear high bits in next word
        words[word + 1] &= ~high_mask
        # Set high bits in next word
        words[word + 1] |= ((value >> (64 - offset)) & high_mask)

class PaletteEntry:
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
        self.data = data # Empty, Uniform, or Palette

    @staticmethod
    def from_voxels(voxels: list) -> 'SectionStorage':
        if len(voxels) != SECTION_VOLUME:
            raise ValueError("Invalid block count")
            
        # Check for Empty or Uniform
        if all(v == 0 for v in voxels):
            return SectionStorage("Empty")
        
        first = voxels[0]
        if all(v == first for v in voxels):
            return SectionStorage(("Uniform", first))
        
        # Build Palette
        entries = []
        indices = [0] * SECTION_VOLUME
        
        for i, v in enumerate(voxels):
            found = False
            for entry in entries:
                if entry.id == v:
                    entry.count += 1
                    indices[i] = entries.index(entry)
                    found = True
                    break
            if not found:
                new_entry = PaletteEntry(v, 1)
                entries.append(new_entry)
                indices[i] = len(entries) - 1
        
        # Shrink entries
        # In Python, list slicing or del can shrink, but here we just rebuild if needed
        # For strict memory, we assume entries is already tight or we trim it.
        
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
*   **Cross-word bit boundary**: When `offset + bits > 64`, the index spans two `u64` words. The implementation must correctly combine the lower bits from the first word and the upper bits from the second word using bitwise OR. Failure to do so results in corrupted data [S1, unofficial].
*   **Palette slot exhaustion**: The sources describe a mechanism where dead slots (count == 0) are reused. If the palette is full and a new ID is encountered, a repack is triggered to increase bit width. However, if the palette is full and no dead slots exist, the system must handle the expansion gracefully [S1, unofficial].
*   **High-entropy palettes**: If a section contains many unique materials (e.g., 4096 unique IDs), the storage overhead (palette entries + bit-packed words) can exceed the 16,384 bytes required by a dense `[u32; 4096]` array. The method does not automatically fall back to dense storage in this case [S1, unofficial].
*   **Coordinate Overflow**: World coordinates are split into section and local parts. Negative coordinates must use floor division (Euclidean division) to correctly map to section indices. Overflow checks are required when computing neighbor offsets [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Read/Write**: $O(1)$ time complexity for accessing a single voxel. It involves bit shifts, masks, and array lookups on 64-bit integers [S1, unofficial].
*   **Memory (Empty/Uniform)**: $O(1)$ bytes (no heap allocation) [S1, unofficial].
*   **Memory (Palette)**: $O(P + \frac{4096 \cdot B}{8})$ bytes, where $P$ is the number of unique entries (each 6 bytes) and $B$ is the bits per index (1–12) [S1, unofficial].

### Measured Performance
On an AMD Ryzen 7 5800H (Windows 11, Rust 1.99):
*   **Empty Section**: 0 B heap storage.
*   **Uniform Section**: 0 B heap storage.
*   **Synthetic Terrain**: 1,048 B heap storage.
*   **Air/Solid Checkerboard**: 528 B heap storage.
*   **4,096 Unique Solid IDs**: 38,912 B heap storage (significantly larger than dense 16,384 B due to palette overhead) [S1, unofficial].

### Strengths and Limitations
*   **Strengths**: Extremely low memory usage for typical voxel worlds (mostly empty or uniform). Fast $O(1)$ access. Compatible with greedy meshing.
*   **Limitations**: High-entropy sections are memory-inefficient. No automatic fallback to dense storage for high entropy. Compaction requires iterating all voxels and repacking bits, which is costly if done frequently [S1, unofficial].

### When to Use
*   Use for voxel worlds with significant empty space or large uniform regions (e.g., terrain, caves).
*   Do not use for highly detailed, high-entropy scenes (e.g., complex organic structures with many materials) where dense storage might be more compact.

## Background

Compact Voxel Section Storage is part of the `voxel-forge` library, an unofficial Rust/Vulkan voxel engine project. The storage and coordinate code were extracted and adapted from this local engine. The implementation focuses on minimizing heap allocations and supporting efficient meshing for real-time rendering. It is not a peer-reviewed academic method but a practical engineering solution for voxel data management [S1, unofficial].

## Sources

[S1] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
