# Adaptive Bit-Packed Voxel Array

Also called: SectionStorage, SectionStorage Adaptive Bit-Packed Voxel Array, Voxel Section

## Overview
The Adaptive Bit-Packed Voxel Array is a memory-efficient storage structure for a fixed-size 16×16×16 voxel section (4096 voxels) that dynamically adjusts its memory layout based on the number of unique voxel IDs present [S1, unofficial]. It supports three states—Empty, Uniform, and Palette—optimizing memory usage for homogeneous regions while supporting sparse updates [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sequence of voxel updates `(position, id)` or a bulk array of 4096 voxel IDs.
- Output: A `SectionStorage` instance containing voxel data accessible via `get`, `set`, `iter`, or `copy_voxels` [S1, unofficial].

**Data Structures:**
The core structure is `SectionStorage`, which contains a `Representation` enum [S1, unofficial]:
1. `Empty`: No heap memory. Represents all `VoxelId::AIR` [S1, unofficial].
2. `Uniform(VoxelId)`: No heap memory. Stores a single `VoxelId` inline [S1, unofficial].
3. `Palette(Palette)`: Uses heap memory [S1, unofficial].
    - `Palette` struct contains:
        - `entries: Vec<PaletteEntry>`: A vector of unique voxel IDs [S1, unofficial].
        - `words: Vec<u64>`: Bit-packed indices [S1, unofficial].
        - `bits: u8`: The number of bits per index [S1, unofficial].

`PaletteEntry` struct:
- `id: VoxelId`: The actual voxel value [S1, unofficial].
- `count: u16`: The number of voxels currently referencing this entry. This enables uniform collapse detection without scanning all 4096 voxels [S1, unofficial].

`SectionMemory` struct (for accounting):
- `inline_bytes: usize`: Memory for `Empty` or `Uniform` states [S1, unofficial].
- `heap_used_bytes: usize`: Actual bytes used in `words` and `entries` [S1, unofficial].
- `heap_capacity_bytes: usize`: Allocated capacity for `words` and `entries` [S1, unofficial].
- `palette_entries: usize`: Number of entries in the palette [S1, unofficial].
- `bits_per_index: u8`: Current bit width [S1, unofficial].

**Constants:**
- `SECTION_VOLUME`: 4096 (16³) [S1, unofficial].
- `SECTION_EDGE`: 16 [S1, unofficial].

**Numbered Steps:**

1. **Initialization:**
    - `SectionStorage::default()` creates an `Empty` section with 0 heap capacity [S1, unofficial].
    - `SectionStorage::filled(id)` creates a `Uniform` section where all 4096 voxels have value `id`, using 0 heap capacity [S1, unofficial].

2. **Bulk Construction (`from_voxels`):**
    - Validate input length is `SECTION_VOLUME` [S1, unofficial].
    - Check if all voxels are identical. If so, return `filled(id)` [S1, unofficial].
    - Otherwise, iterate through all 4096 voxels [S1, unofficial]:
        - Find or create a slot in `entries` for each unique `VoxelId` [S1, unofficial].
        - Increment `count` for the corresponding `PaletteEntry` [S1, unofficial].
        - Store the slot index in a temporary stack-allocated `indices` array [S1, unofficial].
    - Calculate `bits = index_bits(entries.len())` [S1, unofficial].
    - Allocate `words` vector of size `word_count(bits)` [S1, unofficial].
    - Write indices from the temporary array into `words` using bit-packing [S1, unofficial].
    - Set `data` to `Representation::Palette` [S1, unofficial].

3. **Single Update (`set`):**
    - Calculate linear `index` from `LocalPos` [S1, unofficial].
    - If `data` is `Empty` or `Uniform`:
        - If the new `id` matches the current state, return `false` [S1, unofficial].
        - Otherwise, initialize a `Palette` with capacity 4 [S1, unofficial].
        - Add the old value (if `Uniform`) and the new `id` to `entries` [S1, unofficial].
        - Set `bits` to 1 [S1, unofficial].
        - Write the new index to `words` [S1, unofficial].
        - Update `count` for both entries (old value count becomes `SECTION_VOLUME - 1`, new value count becomes 1) [S1, unofficial].
    - If `data` is `Palette`:
        - Read the current slot index `old_slot` from `words` [S1, unofficial].
        - If `palette.entries[old_slot].id == id`, return `false` (no change) [S1, unofficial].
        - If the new `id` is already in `entries`, find its `new_slot` [S1, unofficial].
        - If `new_slot == old_slot`, return `false` [S1, unofficial].
        - If `new_slot != old_slot`:
            - Decrement `count` of `palette.entries[old_slot]` [S1, unofficial].
            - Increment `count` of `palette.entries[new_slot]` [S1, unofficial].
            - Write `new_slot` to `words` at `index` [S1, unofficial].
            - Return `true` [S1, unofficial].
        - If the new `id` is not in `entries`:
            - If `palette.entries[old_slot].count == 1`:
                - Replace `palette.entries[old_slot].id` with the new `id` [S1, unofficial].
                - Return `true` (no dead entry created) [S1, unofficial].
            - Else if there is a "dead" slot (`count == 0`) in `entries`:
                - Reuse the dead slot for the new `id` [S1, unofficial].
                - Update `words` to point to this slot [S1, unofficial].
                - Return `true` [S1, unofficial].
            - Else if `entries.len() < 4096` and `bits < 12`:
                - Add new `id` to `entries` [S1, unofficial].
                - Update `words` to point to the new slot [S1, unofficial].
                - Return `true` [S1, unofficial].
            - Else (palette full or bits limit reached):
                - Behavior is constrained by the 12-bit limit [S1, unofficial].

4. **Batch Update (`set_batch`):**
    - Iterate through a list of `(LocalPos, VoxelId)` tuples [S1, unofficial].
    - Call `set` for each [S1, unofficial].
    - Return the count of successful updates (where `set` returned `true`) [S1, unofficial].

5. **Compaction (`compact`):**
    - Only applicable to `Palette` state [S1, unofficial].
    - Filter `entries` to keep only those with `count > 0` [S1, unofficial].
    - Rebuild the palette mapping:
        - Create a new `entries` vector with live entries [S1, unofficial].
        - Recalculate `bits` based on the new number of entries [S1, unofficial].
        - Reallocate `words` if necessary [S1, unofficial].
        - Rewrite all indices in `words` to point to the new compacted slot indices [S1, unofficial].
    - Return `true` if changes were made [S1, unofficial].

6. **Uniform/Empty Collapse:**
    - After updates or compaction, check if any `PaletteEntry` has `count == SECTION_VOLUME` [S1, unofficial].
    - If so, collapse to `Uniform` state with that `id`, freeing heap memory [S1, unofficial].
    - If all voxels are `AIR`, collapse to `Empty` state with 0 heap capacity [S1, unofficial].

7. **Iteration (`iter` / `copy_voxels`):**
    - For `Empty`: Yield `VoxelId::AIR` for all 4096 indices [S1, unofficial].
    - For `Uniform`: Yield the stored `id` for all 4096 indices [S1, unofficial].
    - For `Palette`: For each index 0 to 4095:
        - Read the bit-packed index from `words` [S1, unofficial].
        - Look up `palette.entries[index].id` [S1, unofficial].
        - Yield the `VoxelId` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SECTION_VOLUME = 4096
SECTION_EDGE = 16
MAX_BITS = 12

def index_bits(num_entries: int) -> int:
    if num_entries <= 1: return 1
    bits = 0
    n = num_entries - 1
    while n > 0:
        bits += 1
        n >>= 1
    return max(1, bits)

def word_count(bits: int) -> int:
    total_bits = SECTION_VOLUME * bits
    return (total_bits + 63) // 64

def write_index(words: list, bits: int, index: int, value: int):
    word_idx = (index * bits) // 64
    bit_offset = (index * bits) % 64
    mask = (1 << bits) - 1
    words[word_idx] &= ~(mask << bit_offset)
    words[word_idx] |= (value & mask) << bit_offset
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
        self.data = None  # Empty, Uniform, or Palette

    @staticmethod
    def default():
        return SectionStorage.filled(0)  # 0 represents VoxelId::AIR

    @staticmethod
    def filled(id):
        s = SectionStorage()
        if id == 0:
            s.data = ("Empty",)
        else:
            s.data = ("Uniform", id)
        return s

    def set(self, position, id):
        index = position.index()  # 0 to 4095
        if self.data[0] == "Empty":
            if id == 0: return False
            self.data = ("Palette", {
                "entries": [{"id": 0, "count": SECTION_VOLUME - 1}, {"id": id, "count": 1}],
                "words": [0],
                "bits": 1
            })
            write_index(self.data[1]["words"], 1, index, 1)
            return True
        
        if self.data[0] == "Uniform":
            current_id = self.data[1]
            if current_id == id: return False
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
            
            if palette["entries"][old_slot]["count"] == 1:
                palette["entries"][old_slot]["id"] = id
                return True
            
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
            
            if len(palette["entries"]) < 4096 and palette["bits"] < MAX_BITS:
                new_slot = len(palette["entries"])
                palette["entries"].append({"id": id, "count": 1})
                write_index(palette["words"], palette["bits"], index, new_slot)
                return True
            
            return False
        
        return False

    def compact(self):
        if self.data[0] != "Palette": return False
        palette = self.data[1]
        live_entries = [e for e in palette["entries"] if e["count"] > 0]
        if len(live_entries) == len(palette["entries"]): return False
        
        old_to_new = {}
        for i, entry in enumerate(live_entries):
            old_to_new[i] = i
        
        new_words = [0] * word_count(index_bits(len(live_entries)))
        for idx in range(SECTION_VOLUME):
            old_slot = read_index(palette["words"], palette["bits"], idx)
            new_slot = old_to_new[old_slot]
            write_index(new_words, index_bits(len(live_entries)), idx, new_slot)
            
        palette["entries"] = live_entries
        palette["words"] = new_words
        palette["bits"] = index_bits(len(live_entries))
        return True

    def iter(self):
        if self.data[0] == "Empty":
            return [0] * SECTION_VOLUME
        if self.data[0] == "Uniform":
            return [self.data[1]] * SECTION_VOLUME
        palette = self.data[1]
        result = []
        for i in range(SECTION_VOLUME):
            slot = read_index(palette["words"], palette["bits"], i)
            result.append(palette["entries"][slot]["id"])
        return result
```

**Parameters:**
- `SECTION_VOLUME`: 4096 [S1, unofficial].
- `SECTION_EDGE`: 16 [S1, unofficial].
- `MAX_BITS`: 12 (supports up to 4096 unique IDs) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Palette Width Limit:** The implementation caps `bits_per_index` at 12 bits. If more than 4096 unique IDs are introduced, the behavior is constrained by this limit [S1, unofficial].
- **Dead Slot Reuse:** Dead slots (`count == 0`) are reused to avoid repacking. This prevents memory fragmentation but requires careful management of the `count` field [S1, unofficial].
- **Boundary Conditions:** Bit-packing must handle word boundaries correctly. Indices 0, 63, 64, 2048, 4095 are critical test cases for bit alignment [S1, unofficial].
- **Uniform Collapse:** The `count` field in `PaletteEntry` allows for O(1) detection of uniform sections, avoiding a full scan of 4096 voxels [S1, unofficial].

## Subtopics

### [Sparse Chunk (Brick) Storage](6680-sparse-chunk-brick-storage.md)
Sparse chunk storage replaces a dense voxel layer with a hash map of fixed-size 16³ chunks to decouple scene resolution from memory usage, storing only non-empty regions. It differs from its parent [Adaptive Bit-Packed Voxel Array](4986-adaptive-bit-packed-voxel-array.md) by operating at a coarser granularity (chunks rather than individual voxels or small bit-packed sections) and from [Octree](5000-octree.md) by avoiding tree traversal for reads, using arithmetic addressing instead.

## Operating characteristics
- **Time Complexity:**
    - `get`: O(1) for Empty/Uniform, O(1) for Palette (bit read + array lookup) [G].
    - `set`: O(1) amortized for Palette (linear scan for new ID is bounded by palette size, but dead slot reuse makes it fast). Worst-case O(N) for palette search if no dead slots and new ID [G].
    - `compact`: O(N) where N is the number of voxels (4096) due to rewriting indices [G].
- **Memory Complexity:**
    - Empty/Uniform: O(1) (no heap) [G].
    - Palette: O(P + V*B/64) where P is palette size, V is 4096, B is bits per index. Max heap usage is for 4096 unique IDs with 12 bits: 4096 * 12 / 8 = 6144 bytes for indices, plus 4096 * sizeof(VoxelId) for palette [G].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Extremely low memory overhead for homogeneous sections (Empty/Uniform). Efficient for sparse updates with limited unique IDs [S1, unofficial].
- **Limitations:** Palette search can be slow if many unique IDs are introduced without dead slots. Compaction is expensive and not automatic [S1, unofficial].
- **When to Use:** Ideal for voxel worlds with large homogeneous areas (sky, ground) and localized changes [G].
- **When Not to Use:** Scenes with high entropy (many unique voxel types per section) where palette overhead outweighs benefits [G].

## Background
Not given in the sources.

## Sources
[S1] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
