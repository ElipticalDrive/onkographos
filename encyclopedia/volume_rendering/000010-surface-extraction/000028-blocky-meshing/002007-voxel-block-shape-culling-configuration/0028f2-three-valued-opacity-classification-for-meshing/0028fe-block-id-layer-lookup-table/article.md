# Block ID Layer Lookup Table

Also called: Block ID to Layer Lookup Table, Lookup Table Optimization for Meshing, MAX_BLOCK_ID Lookup

## Overview
The Block ID Layer Lookup Table is a memory layout optimization that replaces hash-based set lookups with direct array indexing to classify block IDs into mesh layers during the inner meshing loop. It solves the performance bottleneck of iterating over `Set` or `HashSet` structures when processing hundreds of thousands of cells per chunk, by pre-computing a fixed-size typed array indexed by block ID. Unlike hash sets, which incur collision resolution overhead, this method provides $O(1)$ access with minimal constant factors, specifically tailored for the three-valued opacity classification system.

## How it works
**Inputs:**
- `config`: An object containing sets of block IDs for different material types.
  - `config.waterBlockIds`: Set of block IDs classified as water.
  - `config.transparentSolidBlockIds`: Set of block IDs classified as transparent solids (e.g., glass, leaves).
  - Implicitly, all other valid block IDs are classified as `opaque`.
- `MAX_BLOCK_ID`: The maximum valid block ID value, inclusive. This determines the size of the lookup table.

**Outputs:**
- `lookup`: A `Uint8Array` of size `MAX_BLOCK_ID + 1`. Each index corresponds to a block ID, and the value at that index is an integer representing the mesh layer.

**Data Structures:**
- `MESH_LAYERS`: An ordered array of strings defining the layers: `['opaque', 'water', 'transparentSolid']`.
- `lookup`: A `Uint8Array` where:
  - `lookup[blockId] = 0` if `blockId` is opaque.
  - `lookup[blockId] = 1` if `blockId` is water.
  - `lookup[blockId] = 2` if `blockId` is a transparent solid.
  - `lookup[blockId] = 0` (default) if `blockId` is not present in either set (treated as opaque/air).

**Steps:**
1. **Initialize Table:** Create a `Uint8Array` named `lookup` with length `MAX_BLOCK_ID + 1`. Fill all entries with the default value `0` (representing `opaque`).
2. **Populate Water Layer:** Iterate through each `blockId` in `config.waterBlockIds`. Set `lookup[blockId]` to `1` (the index of `'water'` in `MESH_LAYERS`).
3. **Populate Transparent Solid Layer:** Iterate through each `blockId` in `config.transparentSolidBlockIds`. Set `lookup[blockId]` to `2` (the index of `'transparentSolid'` in `MESH_LAYERS`).
4. **Validation:** Ensure all `blockId` values in the configuration sets are $\le MAX\_BLOCK\_ID$ and are valid integers before population to prevent out-of-bounds access [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_layer_lookup(config, max_block_id):
    """
    Builds a direct lookup table for block ID to mesh layer classification.
    
    Args:
        config: Object containing 'waterBlockIds' and 'transparentSolidBlockIds' as iterables.
        max_block_id: Integer, the maximum valid block ID (inclusive).
        
    Returns:
        Uint8Array of size max_block_id + 1.
    """
    # MESH_LAYERS indices: opaque=0, water=1, transparentSolid=2
    # Default is 0 (opaque)
    lookup = [0] * (max_block_id + 1)
    
    # Populate water layer (index 1)
    for block_id in config.waterBlockIds:
        if 0 <= block_id <= max_block_id:
            lookup[block_id] = 1
            
    # Populate transparent solid layer (index 2)
    for block_id in config.transparentSolidBlockIds:
        if 0 <= block_id <= max_block_id:
            lookup[block_id] = 2
            
    return lookup
```

**Parameters:**
- `MAX_BLOCK_ID`: The maximum valid block ID. The table size is exactly `MAX_BLOCK_ID + 1` [S1, unofficial]. Typical values depend on the voxel engine's block ID space; in the reference implementation, this is derived from the specific game or simulation's block ID range [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Out-of-Bounds Access:** If a block ID in the configuration exceeds `MAX_BLOCK_ID`, direct array access will throw an error or corrupt memory. Validation is required before building the table [S1, unofficial].
- **Sparse IDs:** If `MAX_BLOCK_ID` is very large but the number of unique block IDs is small, the table consumes significant memory. This method is optimal when block IDs are dense or the maximum ID is reasonably bounded relative to the memory budget [S1, unofficial].
- **Default Behavior:** IDs not explicitly listed in `waterBlockIds` or `transparentSolidBlockIds` default to `0` (opaque). This matches the semantic requirement that unlisted blocks are solid/opaque [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Build Time:** $O(W + T)$, where $W$ is the number of water block IDs and $T$ is the number of transparent solid block IDs. This is a one-time cost per configuration change.
- **Lookup Time:** $O(1)$ per cell. The inner meshing loop performs a single array access `lookup[blockId]` instead of a hash set lookup.
- **Memory:** $O(MAX\_BLOCK\_ID)$ bytes. The table size is fixed at `MAX_BLOCK_ID + 1` bytes (using `Uint8Array`) [S1, unofficial].

**Measured Performance:**
- In the reference implementation, the inner meshing loop processes approximately 393,216 cells per chunk (6 face passes over a $16 \times 16 \times 256$ chunk) [S1, unofficial].
- The lookup table method is significantly faster than iterating over `Set` or `HashSet` structures because it avoids hash computation and collision resolution overhead [S1, unofficial].
- Benchmarks indicate that the lookup table arm is the most efficient method for the hot path, reducing per-chunk latency compared to native `Set` or custom `HashSet` implementations [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Extremely fast lookup; cache-friendly due to contiguous memory layout; simple implementation.
- **Limitations:** Memory usage scales with `MAX_BLOCK_ID`. If block IDs are sparse or the ID space is huge, this method is inefficient compared to hash-based approaches. Not suitable for dynamic block ID ranges that exceed the pre-allocated size.

**When to Use:**
- Use when block IDs are dense, bounded, and the inner loop performance is critical (e.g., real-time voxel meshing).
- Use when the set of water and transparent solid block IDs is small relative to the total ID space, but the lookup frequency is very high.

**When Not to Use:**
- Use hash sets or other sparse data structures if block IDs are sparse or the maximum ID is extremely large relative to the number of active block types.
- Use if the set of block IDs changes frequently and the build cost outweighs the lookup savings.

## Background
The Block ID Layer Lookup Table was introduced in the `mc-meshing` project by Nerima Games as an optimization for the inner meshing loop in voxel chunk processing [S1, unofficial]. It addresses the performance bottleneck identified in earlier implementations that used `Set` or `HashSet` for block ID classification, which proved too slow for the high-frequency lookups required in the six-pass mask build [S1, unofficial]. The method is part of a broader effort to optimize meshing performance by reducing allocation and improving cache locality [S1, unofficial].

## Sources
[S1] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
