# Adaptive Bit-Packed Voxel Array

Also called: SectionStorage, SectionStorage Adaptive Bit-Packed Voxel Array, Voxel Section

## Overview
The Adaptive Bit-Packed Voxel Array is a memory-efficient storage structure for a fixed-size 16×16×16 voxel section (4096 voxels) that dynamically adjusts its memory layout based on the number of unique voxel IDs present [S1, unofficial]. It supports four states—Empty, Uniform, Palette, and implicitly Dense via Palette limits—optimizing memory usage for homogeneous regions while supporting sparse updates [S1, unofficial]. It differs from its parent [Multi-type Voxel Storage](1104-multi-type-voxel-storage.md) by implementing the adaptive bit-packing logic within a single section, and from [Typed Voxels](1399-typed-voxels.md) by using index-based palette lookups rather than direct value storage for non-uniform data [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sequence of voxel updates (position, ID) or a bulk array of 4096 `VoxelId`s.
- Output: A `SectionStorage` instance containing voxel data accessible via `get`, `set`, `iter`, or `copy_voxels`.

**Data Structures:**
The core structure is `SectionStorage`, which contains a `Representation` enum:
1.  `Empty`: No heap memory. Represents all `VoxelId::AIR`.
2.  `Uniform(VoxelId)`: No heap memory. Stores a single `VoxelId` inline.
3.  `Palette(Palette)`: Uses heap memory.
    -   `Palette` struct contains:
        -   `entries: Vec<PaletteEntry>`: A vector of unique voxel IDs.
        -   `words: Vec<u64>`: Bit-packed indices.
        -   `bits: u8`: The number of bits per index.

`PaletteEntry` struct:
-   `id: VoxelId`: The actual voxel value.
-   `count: u16`: The number of voxels currently referencing this entry. This allows efficient detection of uniform collapse without scanning all 4096 voxels.

`SectionMemory` struct (for accounting):
-   `inline_bytes: usize`: Memory for `Empty` or `Uniform` states.
-   `heap_used_bytes: usize`: Actual bytes used in `words` and `entries`.
-   `heap_capacity_bytes: usize`: Allocated capacity for `words` and `entries`.
-   `palette_entries: usize`: Number of entries in the palette.
-   `bits_per_index: u8`: Current bit width.

**Constants:**
-   `SECTION_VOLUME`: 4096 (16³).
-   `SECTION_EDGE`: 16.

**Numbered Steps:**

1.  **Initialization:**
    -   `SectionStorage::default()` creates an `Empty` section (all `VoxelId::AIR`).
    -   `SectionStorage::filled(id)` creates a `Uniform` section if `id != AIR`, or `Empty` if `id == AIR`. Heap capacity is 0.

2.  **Bulk Construction (`from_voxels`):**
    -   Validate input length is `SECTION_VOLUME`.
    -   Check if all voxels are identical. If so, return `filled(id)`.
    -   Otherwise, iterate through all 4096 voxels:
        -   Find or create a slot in `entries` for each unique `VoxelId`.
        -   Increment `count` for the corresponding `PaletteEntry`.
        -   Store the slot index in a temporary stack-allocated `indices` array.
    -   Calculate `bits = index_bits(entries.len())`.
    -   Allocate `words` vector of size `word_count(bits)`.
    -   Write indices from the temporary array into `words` using bit-packing.
    -   Set `data` to `Representation::Palette`.

3.  **Single Update (`set`):**
    -   Calculate linear `index` from `LocalPos`.
    -   If `data` is `Empty` or `Uniform`:
        -   If the new `id` matches the current state, return `false`.
        -   Otherwise, initialize a `Palette` with capacity 4.
        -   Add the old value (if `Uniform`) and the new `id` to `entries`.
        -   Set `bits` to 1.
        -   Write the new index to `words`.
        -   Update `count` for both entries (old value count becomes `SECTION_VOLUME - 1`, new value count becomes 1).
    -   If `data` is `Palette`:
        -   Read the current slot index `old_slot` from `words`.
        -   If `palette.entries[old_slot].id == id`, return `false` (no change).
        -   If the new `id` is already in `entries`, find its `new_slot`.
        -   If `new_slot == old_slot`, return `false`.
        -   If `new_slot != old_slot`:
            -   Decrement `count` of `palette.entries[old_slot]`.
            -   Increment `count` of `palette.entries[new_slot]`.
            -   Write `new_slot` to `words` at `index`.
            -   Return `true`.
        -   If the new `id` is not in `entries`:
            -   If `palette.entries[old_slot].count == 1`:
                -   Replace `palette.entries[old_slot].id` with the new `id`.
                -   Return `true`. (No dead entry created).
            -   Else if there is a "dead" slot (count == 0) in `entries`:
                -   Reuse the dead slot for the new `id`.
                -   Update `words` to point to this slot.
                -   Return `true`.
            -   Else if `entries.len() < 4096` and `bits < 12`:
                -   Add new `id` to `entries`.
                -   Update `words` to point to the new slot.
                -   Return `true`.
            -   Else (palette full or bits limit reached):
                -   Behavior is constrained by the 12-bit limit. The implementation may fail to add the new ID or require compaction first. [S1, unofficial]

4.  **Batch Update (`set_batch`):**
    -   Iterate through a list of `(LocalPos, VoxelId)` tuples.
    -   Call `set` for each.
    -   Return the count of successful updates (where `set` returned `true`).

5.  **Compaction (`compact`):**
    -   Only applicable to `Palette` state.
    -   Filter `entries` to keep only those with `count > 0`.
    -   Rebuild the palette mapping:
        -   Create a new `entries` vector with live entries.
        -   Recalculate `bits` based on the new number of entries.
        -   Reallocate `words` if necessary.
        -   Rewrite all indices in `words` to point to the new compacted slot indices.
    -   Return `true` if changes were made.

6.  **Uniform/Empty Collapse:**
    -   After updates or compaction, check if any `PaletteEntry` has `count == SECTION_VOLUME`.
    -   If so, collapse to `Uniform` state with that `id`, freeing heap memory.
    -   If all voxels are `AIR`, collapse to `Empty` state.

7.  **Iteration (`iter` / `copy_voxels`):**
    -   For `Empty`: Yield `VoxelId::AIR` for all 4096 indices.
    -   For `Uniform`: Yield the stored `id` for all 4096 indices.
    -   For `Palette`: For each index 0 to 4095:
        -   Read the bit-packed index from `words`.
        -   Look up `palette.entries[index].id`.
        -   Yield the `VoxelId`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SECTION_VOLUME = 4096
SECTION_EDGE = 16
MAX_BITS = 12

def index_bits(num_entries: int) -> int:
    if num_entries <= 1: return 1
    if num_entries <= 2: return 1
    # Calculate ceiling of log2(num_entries)
    bits = 0
    n = num_entries - 1
    while n > 0:
        bits += 1
        n >>= 1
    return max(1, bits)

def word_count(bits: int) -> int:
    # Total bits needed
    total_bits = SECTION_VOLUME * bits
    # Number of 64-bit words
    return (total_bits + 63) // 64

def write_index(words: list, bits: int, index: int, value: int):
    # Write 'value' into 'words' at bit position 'index * bits'
    word_idx = (index * bits) // 64
    bit_offset = (index * bits) % 64
    mask = (1 << bits) - 1
    # Clear existing bits
    words[word_idx] &= ~(mask << bit_offset)
    # Set new bits
    words[word_idx] |= (value & mask) << bit_offset
    # Handle overflow to next word if bits + offset > 64
    if bit_offset + bits > 64:
        words[word_idx + 1] &= ~(mask >> (64 - bit_offset))
        words[word_idx + 1] |= (value & mask) >> (64 - bit_offset)

def read_index(words: list, bits: int, index: int) -> int:
    word_idx = (index * bits) // 64
    bit_offset = (index * bits) % 64
    mask = (1 << bits) - 1
    val = (words[word_idx] >> bit_offset) & mask
    if bit_offset + bits > 64:
        val |= (words[word_idx + 1] & (mask >> (64 - bit_offset))) << (64 - bit_offset)
    return val

class SectionStorage:
    def __init__(self):
        self.data = None # Empty, Uniform, or Palette

    @staticmethod
    def default():
        return SectionStorage.filled(VoxelId.AIR)

    @staticmethod
    def filled(id):
        s = SectionStorage()
        if id == VoxelId.AIR:
            s.data = "Empty"
        else:
            s.data = ("Uniform", id)
        return s

    def set(self, position, id):
        index = position.index() # 0 to 4095
        if self.data == "Empty":
            if id == VoxelId.AIR: return False
            # Transition to Palette
            self.data = ("Palette", {
                "entries": [{"id": VoxelId.AIR, "count": SECTION_VOLUME - 1}, {"id": id, "count": 1}],
                "words": [0],
                "bits": 1
            })
            write_index(self.data[1]["words"], 1, index, 1)
            return True
        
        if self.data[0] == "Uniform":
            current_id = self.data[1]
            if current_id == id: return False
            # Transition to Palette
            self.data = ("Palette", {
                "entries": [{"id": current_id, "count": SECTION_VOLUME - 1}, {"id": id, "count": 1}],
                "words": [0],
                "bits": 1
            })
            write_index(self.data[1]["words"], 1, index, 1)
            return True

        if self.data[0] == "Palette":
            palette = self.data[1]
            old_slot = read_index(palette["words"], palette["bits"], index)
            old_id = palette["entries"][old_slot]["id"]
            
            if old_id == id: return False
            
            # Find new slot
            new_slot = None
            for i, entry in enumerate(palette["entries"]):
                if entry["id"] == id:
                    new_slot = i
                    break
            
            if new_slot is not None:
                if new_slot == old_slot: return False
                palette["entries"][old_slot]["count"] -= 1
                palette["entries"][new_slot]["count"] += 1
                write_index(palette["words"], palette["bits"], index, new_slot)
                return True
            
            # New ID
            if palette["entries"][old_slot]["count"] == 1:
                # Replace in place
                palette["entries"][old_slot]["id"] = id
                return True
            
            # Check for dead slot
            dead_slot = None
            for i, entry in enumerate(palette["entries"]):
                if entry["count"] == 0:
                    dead_slot = i
                    break
            
            if dead_slot is not None:
                palette["entries"][dead_slot]["id"] = id
                palette["entries"][dead_slot]["count"] = 1
                write_index(palette["words"], palette["bits"], index, dead_slot)
                return True
            
            # Add new entry
            if len(palette["entries"]) < 4096 and palette["bits"] < MAX_BITS:
                new_slot = len(palette["entries"])
                palette["entries"].append({"id": id, "count": 1})
                write_index(palette["words"], palette["bits"], index, new_slot)
                return True
            
            # Limit reached
            return False
        
        return False

    def compact(self):
        if self.data[0] != "Palette": return False
        palette = self.data[1]
        live_entries = [e for e in palette["entries"] if e["count"] > 0]
        if len(live_entries) == len(palette["entries"]): return False
        
        # Rebuild mapping
        old_to_new = {}
        for i, entry in enumerate(live_entries):
            old_to_new[i] = i # Simplified: assumes we just remove dead ones
        
        # This is a simplification; real implementation needs to remap indices
        # For brevity, we assume a full rebuild is done by scanning and rewriting
        # In practice, this involves creating a new palette and rewriting all words
        
        # Placeholder for complex remapping logic
        # ...
        return True
```

**Parameters:**
-   `SECTION_VOLUME`: 4096.
-   `SECTION_EDGE`: 16.
-   `MAX_BITS`: 12 (supports up to 4096 unique IDs).

**Edge Cases and Pitfalls:**
-   **Palette Width Limit:** The implementation caps `bits_per_index` at 12 bits. If more than 4096 unique IDs are introduced, the behavior is constrained. The source does not specify if it fails, truncates, or requires external handling [S1, unofficial].
-   **Dead Slot Reuse:** Dead slots (count == 0) are reused to avoid repacking. This prevents memory fragmentation but requires careful management of the `count` field.
-   **Boundary Conditions:** Bit-packing must handle word boundaries correctly. Indices 0, 63, 64, 2048, 4095 are critical test cases for bit alignment [S1, unofficial].
-   **Uniform Collapse:** The `count` field in `PaletteEntry` allows for O(1) detection of uniform sections, avoiding a full scan of 4096 voxels.

## Subtopics

### [Sparse Chunk (Brick) Storage](6680-sparse-chunk-brick-storage.md)
Sparse chunk storage replaces a dense voxel layer with a hash map of fixed-size 16³ chunks to decouple scene resolution from memory usage, storing only non-empty regions. It differs from its parent [Adaptive Bit-Packed Voxel Array](4986-adaptive-bit-packed-voxel-array.md) by operating at a coarser granularity (chunks rather than individual voxels or small bit-packed sections) and from [Octree](5000-octree.md) by avoiding tree traversal for reads, using arithmetic addressing instead.

## Operating characteristics
-   **Time Complexity:**
    -   `get`: O(1) for Empty/Uniform, O(1) for Palette (bit read + array lookup).
    -   `set`: O(1) amortized for Palette (linear scan for new ID is bounded by palette size, but dead slot reuse makes it fast). Worst-case O(N) for palette search if no dead slots and new ID.
    -   `compact`: O(N) where N is the number of voxels (4096) due to rewriting indices.
-   **Memory Complexity:**
    -   Empty/Uniform: O(1) (no heap).
    -   Palette: O(P + V*B/64) where P is palette size, V is 4096, B is bits per index. Max heap usage is for 4096 unique IDs with 12 bits: 4096 * 12 / 8 = 6144 bytes for indices, plus 4096 * sizeof(VoxelId) for palette.
-   **Measured Performance:** Not given in the sources.
-   **Strengths:** Extremely low memory overhead for homogeneous sections (Empty/Uniform). Efficient for sparse updates with limited unique IDs.
-   **Limitations:** Palette search can be slow if many unique IDs are introduced without dead slots. Compaction is expensive and not automatic.
-   **When to Use:** Ideal for voxel worlds with large homogeneous areas (sky, ground) and localized changes.
-   **When Not to Use:** Scenes with high entropy (many unique voxel types per section) where palette overhead outweighs benefits.

## Background
The Adaptive Bit-Packed Voxel Array is part of the Voxel Forge project, an open-source voxel engine implementation [S1, unofficial]. It was developed to provide efficient memory management for voxel sections in real-time applications, balancing the need for fast access with low memory footprint. The structure is designed to handle the dynamic nature of voxel editing, where sections frequently transition between empty, uniform, and sparse states.

## Sources
[S1] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
